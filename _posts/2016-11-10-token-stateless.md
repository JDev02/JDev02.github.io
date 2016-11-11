---
layout: post
title: Petición segura al servidor Stateless con token de verificación.
subtitle: ...Sin uso de memoria, session, ni persistencia... cómo?
---

Hola, este es mi primer post.
Cómo mencioné en About Me: “si no tienes blog no existes”...

Es por eso que aquí vamos, espero tomarle el gusto a esto del blog, y seguir aportando y ayudando con experiencias día a día.
<br>Hoy voy a comentar como resolver de una forma rápida y sencilla un problema que suele suceder cuando se intenta añadir algo de seguridad a peticiones get/post sensibles/delicadas de una App en MVC .NET.<br>
Normalmente cuando se quiere desplegar un documento pdf, imagen u otro hacemos la petición directa al archivo, por lo cual no existe problema. <br>
Muchas veces el requerimiento es agregarle un poco de seguridad a estas acciones, y la primera solución que sale al aire es; pedir el mismo documento, pero a una suerte de api, donde podríamos tener app/resource/{id}  Allí podríamos hacer un par de checkeos de sesion actual del usuario, seguridad, si tiene acceso para ese archivo, etc, etc. <br>
El problema surge cuando tienes un escenario similar en una web (que para aclarar, esta muy mal diseñada, tiene uso de Session,TempData,etc por todo su código, etc) y debes controlar el acceso a esos GET/POST. <br>
Entonces, podría darse que el usuario tenga todos los permisos para ver documentos y sepa que haciendo app/resource/N obtendria un documento X, y, es eso lo que queremos controlar.

Una solución que habitualmente se ve es; antes de hacer GET ir al servidor e indicarle que "el usuario X(guardado típicamente en session... muy mal, pero bue...) hará una petición a este documento, por lo cual dale acceso". Así se controla que ese GET tenga que pasar por algún proceso anterior de la appweb y dar un ticket de autorización (guardado normalmente en memoria para su posterior checkeo y validación) y luego acceder al archivo y eliminar el ticket de memoria.<br>
<b>Por qué no me gusta?</b><br> porque se debe usar algún sistema de persistencia, con lo que perdemos el servidor web stateless y la web se vuelve poco autosuficiente. Normalmente guardarias un "ticket" en TempData, Session, que al ser usado es eliminado... de donde? de la memoria del servidor... Si no es en memoria será en algún txt, tabla, json, etc...


Mejorándolo...
--------------

1.-En un ataque de iluminación dices, "ah!, claro, le agrego un parámetro encriptado que sea 'valido' y si es válido le permito ver el archivo, genial, esa es la solución"... <br>Luego el usuario se da cuenta que puede cambiar el id del documento, manteniendo el "token", y podrá revisar los otros documentos perfectamente, ya que este token sigue siendo "valido"

2.-Bueno... ya sé que la primera versión no es de las mejores, asi que en otro ataque de genialidad, te dices; "Le generamos un token valido para la petición de ese documento! y todo queda ok!" el jefe queda feliz y todo sigue su camino. <br>
<b>Por qué no me gusta?</b><br> ...A las semanas se dan cuenta que los usuarios se están compartiendo los links por correo y pueden acceder a los documentos.... el link tiene un token "valido" para ese documento.

Versión final:
--------------

Aquí explicaré como resolver este problema (aclaro que la aplicación en la que trabajé ya esta muy manoseada, con muy malas prácticas, sin nada de DI... (por ahora estoy intentando quitar del todo la Session, para lo cual cree una capa de Tenant que ya mostrare en un siguiente post) y existen mejores soluciones trabajando <b>bien</b> con http.)

La solución trabajará colaborando entre POST y GET.

En un controller vamos a definir un método que será él que se encargará de autorizar y proveer de dichos tokens/tickets

```cs
        [HttpPost]
        public ActionResult Authorize(int id)
        {
            //checkeos varios de session seguridad, integridad etc...

            
            //el parametro id ya puede venir 
            var result = new LinkResourceDescriptor()
            {
                HttpAction = Common.Response.Constants.HttpMethodTypes.Get,
                HttpMethodType = Common.Response.Constants.HttpMethodTypes.Get,
                Rel = "self",
                Href =
                    string.Format("{0}/{1}?token={2}", Url.Action("Ver", "Controler"), id,
                        AuthorizationTokenHelper.CreateFor(id.ToString())),
                Title = string.Format("Documento {0}", id)
            };

            return Json(result, JsonRequestBehavior.AllowGet);
        } 
```

Nota: LinkResourceDescriptor es una clase propia que utilizo para retornar un JSON con información al navegador.

Como se ve, la clave está en la clase estática AuthorizationTokenHelper y su método CreateFor. 

```cs
        public static string CreateFor(string key)
        {
            return CreateTokenFor(key);
        }

		private static string CreateTokenFor(string text)
        {
            string prefix = CreatePrefix(text);
            string suffix = CreateSuffix(text);
            string currentUser = CreatePrefix(App.Web.Tenant.Current.Usuario);
            string currentEmpresa = CreateSuffix(App.Web.Tenant.Current.Empresa);

            return string.Format("{0}{1}{2}{3}{4}{5}{6}", prefix, Delimiter, suffix, Delimiter, currentUser, Delimiter, currentEmpresa);
        }
```

Es aquí donde tienes que echar a volar la imaginación y crear tu forma de creación/validación.<br> En este caso básicamente se esta creando un token único que le pertenece a esa id de petición, donde CreatePrefix (nombre totalmente confuso, pero es como se estaba usando) es usado para encriptar de una forma y CreateSuffix para encriptar lo mismo, pero de otra forma. Si te fijas, a simple vista veras que la aplicación es MultiTenant por lo cual con App.Web.Tenant.Current.Usuario obtienes el usuario que esta generando el token, y con Current.Empresa, su empresa. Luego se crea y retorna el token final. <br>
Como se puede ver, la key que pasas para crear el token puede ser la que se te antoje, esta <b>ya</b> podría venir encriptada (podría ser un simple base64, eso lo defines tu), o ser simplemente el id real.

El metodo CreateTokenFor retornará algo similar a 
criptoPrefix_CriptoSufix_CriptoUser_CriptoEmpresa
xxxAsasasXXX_4124aTyaaas_Assaetrasa_KamepoyayossU

Nota: Existe otro método que solo se encarga de crear un token, su contra parte es que genera el token, y si luego checkeas es válido, pero para nuestro caso necesitamos un createFor(podriamos apoyarnos de Create, pero a veces es mejor ser explicito, aunque el Create y CreateFor hagan algo muy similar, mejor separemos la responsabilidad y definición de los métodos) y ValidateFor, porque necesitamos que valide para esa key y token en especial, no para indicarnos si es válido o no.


Finalmente
---------- 

```cs
        [HttpGet]
        public ActionResult Visualiza(int id, string token)
        {
            //checkeos varios de sesion seguridad, integridad etc...

            bool isValid = AuthorizationTokenHelper.IsValidFor(id.ToString(), token);
            if (isValid)
            {
                return DocumentView(id);
            }

            //excepción o el código que se desee indicar.
        }
```


Con IsValidFor pasamos el id y el token que queremos validar.

El código es el siguiente:

```cs
        private static bool IsValidFor(string key, string token)
        {
        	//se descompone el token
            string tokePrefix = GetTokenPrefix(token);
            string tokeSuffix = GetTokenSuffix(token);
            string tokeUser = GetTokenUser(token);
            string tokeEmpresa = GetTokenEmpresa(token);

            //convertimos el token encriptado en el valor original (el valor cuando se mando a crear)-->desencriptar primer metodo de encripatacion
            var originalKeyPrefix = RestoreByPrefix(tokePrefix);
            //convertimos el token encriptado en el valor original (el valor cuando se mando a crear)-->desencriptar segundo metodo de encripatacion
			var originalKeySuffix = RestoreBySuffix(tokeSuffix);

            //en este punto originalKeyPrefix deberia ser igual a originalKeySuffix

            //restauramos el user que genero el token
            var tokenCreatedByUser = RestoreByPrefix(tokeUser);
            //restauramos la empresa para la cual fue generado
            var tokenCreatedByUserEmpresa = RestoreBySuffix(tokeEmpresa);

            var currentUser = App.Web.Tenant.Current.Usuario;
            var currentEmpresa = App.Web.Tenant.Current.Empresa;

            return originalTextPrefix == originalTextSuffix && (tokenCreatedByUser == currentUser) && (tokenCreatedByUserEmpresa == currentEmpresa) && key == originalTextPrefix && key == originalTextSuffix ;
        }
```

El código esta comentado explicando su funcionamiento.<br>
Para terminar retornamos los checkeos necesarios para saber si es un token valido para ese GET en específico,
luego se compara que las variables originalKeyPrefix y originalKeySuffix obtenidas sean identicas (recordar que su contenido original es exactamente igual, la única diferencia es su metodo de encriptación) también comparamos el usuario y la empresa y finalmente comparamos el id (key) para el cual queremos chekear si el token es valido, ya que puede que originalKeyPrefix y originalKeySuffix sean identicos, pero podrian no ser identicos y validos para el id el cual se esta comprobando (la logica del método IsValid, es idéntica solo que IsValid <b>no</b> comprueba contra un key 'maestra', solo verifica que sea un token válido)

Antes de terminar, mencionar que al generar el token, usé CreatePrefix y CreateSuffix para encriptar el mismo dato de formas distintas, ya que así le puedes dar mas seguridad a tu token, si alguien descubre como encriptar el primer trozo de tu token(keyencriptada_) podría generar tokens validos para Id's en las peticiones... aquí el trabajo es mayor pues, el atacante tendría que lograr encriptar la misma key de dos formas distintas (en realidad primero tendría que detectar el patron de comportamiento de como se conforma y crea tu token -el mas facil seria usuario y empresa-)

Para terminar, y una posible versión 2, podrias montarte una solución muy similar, donde no tengas que pasar explicito string id, string token, y solo pasar el token/ticket que sea autosuficiente, donde este te retorne un objeto que contenta la key original (en este caso el valor del id de la petición) su estado, y lo que necesites.

Hasta aquí llega este post, (ya subiré el código a github) como se puede ver, siempre hay soluciones para no usar session y crear recursos autosuficientes sin tener que por ejemplo crear un lista en memoria en el servidor (ya sea es una variable static o usando la nunca querida session) para saber que alguien te hará un get/post de algo, o simplemente para autorizarlo. (¡NO USES SESSION!).
<br>
En el próximo post explicare como crear una capa Tenant para guarda el dato mas usado por muchos desarrolladores (user, id, name, customer) y luego como incluirle una capa de cache al mismo Tenant, sin usar session.<br>
Cuesta mucho dejar de usar Session/TempData, etc por lo cómoda, pero, si queremos aplicaciones escalables debemos intentarlo.

Saludos!<br>
John.