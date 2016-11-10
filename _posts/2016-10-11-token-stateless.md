---
layout: post
title: Peticion al servidor Stateless con token de verificación.
subtitle: ...Sin uso de memoria, session, ni persistencia... cómo?
---

Hola, este es mi primer post.
Cómo menciono en About Me: “si no tienes blog no existes”...

Es por eso que aquí vamos, espero tomarle el gusto a esto del blog, y seguir aportando y ayudando con experiencias día a día.
 Hoy voy a comentar como resolví de una forma rápida y sencilla un problema que teniamos cuando se le intento añadir algo de seguridad a peticiones get/post sensibles/delicadas de una App en MVC 3.
 Normalmente cuando queremos desplegar un documento pdf, imagen u otro hacemos la peticion directa al archivo, por lo cual no existe problema. 
Muchas veces nos piden agregarle un poco de seguridad a estas acciones, y la primera solución que sale al aire es; pedir el mismo documento, pero a una suerte de api, donde podriamos tener app/resource/{id} allí podriamos hacer un par de checkeos de session actual del usuario, seguridad, si tiene acceso para ese archivo, etc, etc. El problema surge cuando tienes un escenario similar en una web (que para aclarar, esta muy mal diseñada, tiene uso de Session,TempData,etc por todo su código, etc) y debes controlar el acceso a esos GET/POST. 
Entonces, podria darse que el usuario tenga todos los permisos para ver documentos y sepa que haciendo app/resource/N obtendria un documento X, y, es eso es lo que queremos controlar.

Una solución que he visto es; antes de hacer este GET ir al servidor e indicarle que "el usuario X(guardado tipicamente en session... muy mal, pero bue...) hará una petición a este documento, por lo cual dale acceso", asi controlan que ese GET tenga que pasar por algun proceso anterior de la appweb y dar ticket de autorizacion (guardado normalmente en memoria para su posteriro checkeo y validacion) y luego acceder al archivo y eliminar el tikect de memoria.
Por qué no me gusta? porque se debe usar algun sistema de persistencia, con lo que perdemos el servidor web stateless y la web se vuelve poco autosuficiente. Normalmente guardarias un "ticket" en TempData, Session, que al ser usado es eliminado... de donde? de la memoria del servidor... Si no es en memoria será en algun txt, tabla, json, etc...


Mejorandolo...
--------------

1.-En un ataque de iluminación dices, "ah!, claro, le agrego un parametro encriptado que sea 'valido' y si es valido le permito ver el archivo, genial, esa es la solucion"... Luego el usuario se da cuenta que puede cambiar el id del documento, manteniendo el "token", y podra revisar los otros documentos perfectamente, ya que este token sigue siendo "valido"

2.-Bueno, ya sé que la primera versión no es de las mejores, asi que en otro ataque de genialidad, te dices; "Le generamos un token valido para la petición de ese documento! y todo queda ok, el jefe queda feliz y todo sigue su camino. 
Por qué No me gusta? Simple, a las semanas se dan cuenta que los usuarios se estan compartiendo los links por correo y pueden acceder a los documentos, pues el link tiene un token valido para ese documento.

Version final:

Aqui les voy a explicar como resolví este problema (aclaro que la aplicación ya esta muy manoseada, con muy malas practicas, sin nada de DI... (por ahora estoy intentando quitar del todo la Session, para lo cual cree una capa de Tenant que ya mostrare en un siguiente post) y existen mejores soluciones trabajando bien con http a nivel cabeceras, etc...)

Primero vamos a trabajar colaborando entre POST y GET.

En un controller vamos a definir un metodo que sera él que se encargará de autorizar y proveer de dichos token/tickets

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
                    string.Format("{0}/{1}?token={2}", Url.Action("Ver", "NotaDeVenta"), id,
                        AuthorizationTokenHelper.CreateFor(id.ToString())),
                Title = string.Format("Nota de Venta {0}", id)
            };

            return Json(result, JsonRequestBehavior.AllowGet);
        } 
```

Nota: LinkResourceDescriptor es una clase propia que utilizo para retornar información al navegador.

Como ves, la clave esta en la clase estatica AuthorizationTokenHelper y su método, CreateFor. 

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

Es aquí donde tienes que echar a volar la imaginación y crear tu forma de creación/validación. En este caso básicamente estoy creando un token único que le pertenece a esa id de peticion, donde CreatePrefix (nombre totalmente confuso, pero es como se estaba usando) es usado para encriptar de una forma y CreateSuffix para encriptar lo mismo, pero de otra forma. Si te fijas, a simple vista veras que la aplicación es MultiTenant por lo cual con App.Web.Tenant.Current.Usuario obtienes el usuario que esta generando el token, y con Current.Empresa, su empresa. Luego creamos y retornamos el token final. 
Como peudes ver la key que pases para crear el token puede ser la que se te antoje, esta YA podria venir encriptada(podría ser un simple base64, eso lo defines tu), o ser simplemente el id real.

El metodo CreateTokenFor retornará algo similar a 
criptoPrefix_CriptoSufix_CriptoUser_CriptoEmpresa
xxxAsasasXXX_4124aTyaaas_Assaetrasa_KamepoyayossU

Nota: Existe otro método que solo se encarga de crear un token, su contra parte es que genera el token, y si luego checkeas es válido, pero para nuestro caso necesitamos un createFor(Create serviría, pero a veces es mejor ser explicito, aunque el Create y createFor hagan algo muy similar, mejor separemos la responsablidad y definición de los metodos) y ValidateFor, porque necesitamos que válide para esa key y token en especial, no para indicarnos si es valido o no.


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

En el código esta comentado explicando su funcionamiento.
Para terminar retornamos los checkeos necesarios para saber si es un token valido para ese GET en especifico,
entonces luego comparamos que las variables originalKeyPrefix y originalKeySuffix obtenidas sean identicas (recordemos que su contenido original es exactamente igual, solo que con distintos metodos de encriptación) tambien comparamos el usuario y la empresa y finalmente comparamos el id (key) para el cual queremos chekear si el token es valido, ya que puede que originalKeyPrefix y originalKeySuffix sean identicos, pero podrian no ser identicos y validos para el id el cual se esta comprobando (el método IsValid, es identico solo que NO comprueba contra un key 'maestra', solo verifica que sea un token valido)

Antes de terminar, mencionar que al generar el token, usé CreatePrefix y CreateSuffix para encriptar el mismo dato de formas distintas, ya que así le puedes dar mas seguridad a tu token, si alguien descubre como encriptar el primer trozo de tu token(keyencriptada_) podria generar tokens validos para Id's en las peticiones... aquí el trabajo es mayor pues, el atacante tendría que lograr encriptar la misma key de dos formas distintas (en realidad primero tendría que detectar el patron de comportamiento de como se conforma y crea tu token -el mas facil seria usuario y empresa-)

Para terminar, y una posible version 2, podrias montarte una solución muy similar, donde no tengas que pasar explicito string id, string token, y solo pasar el token/ticket que sea autosuficiente, donde este te retorne un objeto que contenta la key original (en este caso el valor del id de la petición) su estado, y lo que necesites.

Hasta aquí llega este post, (ya subiré el codigo a github) como podemos ver, siempre hay soluciones para no usar session y crear recorsos autosuficientes sin tener que por ejemplo crear un lista en memoria en el servidor (ya sea es una variable static o usando la nunca querida session) para saber que alguien te hará un get/post de algo, o simplemente para autorizarlo. (¡NO USES SESSION!).
En el próximo post explicare como crear una capa Tenant para guarda el dato mas usado por muchos desarrolladores (user, id, name, customer) y luego como incluirle una capa de cache al mismo Tenant, sin usar session.
Cuesta mucho dejar de usarla por lo comoda, pero, si queremos aplicaciones escalables debemos intentarlo.

Saludos!
John.