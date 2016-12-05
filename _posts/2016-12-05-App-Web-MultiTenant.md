---
layout: post
title: Aplicación Web Stateless MultiTetant Parte 1
subtitle: MultiTetant, Stateless. Sin Session, y datos minimos de autorización (Empresa, User, etc)
---

Hola, hoy mostraré como crear una suerte de Tenant Factory para aplicaciones web multi tenant.
En esta oportunidad la aplicación afectada tenia un Front End bastante pulido, pero a pesar de eso, el back respondia de forma casi sincronica, por lo cual se comenzó a refactorizar.

En mi experiencia desarrollando aplicaciones web, siempre me he encontrado con aplicaciones que son multi tenant (aka multi empesa, multi inquilino...) 
También la mayoría de los proyectos con los que he tenido que trabajar estan hechos basicamente apoyandose de la Session (System.Web) con la cual manejan lo datos de Session de cada usuario, guardando como minimo estos dos datos:
1.-Usuario
2.-Empresa

Otros mas osados guardan directamente connectionString...

En primer lugar, ya he hablado sobre lo mal que esta usar Session en aplicaciones web, y lo que significa mantener datos perssitentes de configuración/navegación en la memoria de la aplicación (por ejemplo ante un reinicio de la app, todos los usaurios perderian lo que están haciendo en ese momento... su Session ya no sería valida para después del reinicio... se podría usar otro provider de session, pero aun asi no lo recomiendo. Las peticiones a la Session se bloquean (es un objeto "seguro"), por lo cual no consigues mucho creando un front-end asincrono y elegante, si tu back es sincrono (una suerte de fifo)).

La idea es que ésta misma solución la puedan llevar a una dll distinta a la de la aplicación, para luego poder compartir con distintas Areas u otras webs de su misma propiedad, también sería apropiado aplicar algo de SOLID, para convertirla en una interfaz, y mejorar algunos aspectos para después si se requiere poder inyectar. Pero vamos, para el ejemplo se usará un metodo estatico que es el encargado de crear/validar.

Los datos serán guardados en cookies, ya que el front usaba peticiones Ajax, sin configurar un metodo/token de autentificacion (ni hablar de cabeceras o jwt)

En la aplicación que tocó refactorizar existian basicamente dos propiedades que eran utilizadas para identificar la empresa donde estaba logeado el usuario actual, una era el usuario, y otra para la empresa (Anteriormente se guardaba en Session["Usuario"], Session["Empresa"]....) Estas solo disponen de Get, el set es privado, y en eralidad no existe, ya que la única forma de crearlos es a traves del metodo Tenant.Create({...})

```cs
        public string Empresa
        {
            get
            {
                //en este punto es donde sería bueno inyectar un dataProvider...
                string currentEmpresa = CookieUtils.GetCurrentCookieCustomerValue();
                if (currentEmpresa == null)
                {
                    throw new NullReferenceException("currentEmpresa");
                }
                //si el tenant usa algun tipo de encriptacion, en este punto se debe desencriptar la cookie.
                return currentEmpresa;
            }
        }

        public string Usuario
        {
            get
            {
                //en este punto es donde sería bueno inyectar un dataProvider...
                string currentUser = CookieUtils.GetCurrentCookieUserValue();
                if (currentUser == null)
                {
                    throw new NullReferenceException("currentUser");
                }
                //si el tenant usa algun tipo de encriptacion, en este punto se debe desencriptar la cookie.
                return currentUser;
            }
        }
```


Luego creamos un constructor privado para evitar que alguien en un lugar equivocado logee más veces a un usuario, asi le delegamos la responsabilidad al metodo estatic Create. Así:


```cs
        public static Tenant Current
        {
            get
            {
                return GetCurrent();
            }
        }


        public static Tenant Create(string empresa, string usuario)
        {
            //validaciones
            if (empresa == null) throw new ArgumentNullException("empresa");
            if (usuario == null) throw new ArgumentNullException("usuario");

            string tenantCustomerCookieValue = CookieUtils.GetCurrentCookieCustomerValue();
            bool empresaCargada = !string.IsNullOrEmpty(tenantCustomerCookieValue);
            var tenantUserCookieValue = CookieUtils.GetCookieValue(TenantWebExample.TenantConstants.CookieUser);
            bool userCargado = !string.IsNullOrEmpty(tenantUserCookieValue);
            if (empresaCargada || userCargado)
            {
                throw new Exception(string.Format("El usuario {0} ya se encuentra logeado en la empresa {1}", tenantCustomerCookieValue ?? string.Empty, tenantUserCookieValue ?? string.Empty));
            }

            //aqui pueden encriptar los nombres de la cookie y sería bueno inyectar un manager para el store de los datos...
            CookieUtils.CreateCookie(TenantWebExample.TenantConstants.CookieEmpresa, empresa);
            CookieUtils.CreateCookie(TenantWebExample.TenantConstants.CookieUser, usuario);

            //Fluent por si se quisiera hacer algo con el tenant actual creado....
            return new Tenant();
        }
```


Proveemos de algunos datos de utilidades para ciertos escenarios:



```cs
        public static bool IsLogged()
        {
            //en este punto es donde sería bueno inyectar un dataCookieProvider...
            return !string.IsNullOrEmpty(CookieUtils.GetCurrentCookieCustomerValue());
        }
```

Otro metodo para hacer logout:


```cs
        public static void LogOut()
        {
            var cookieList = TenantConstants.CookieList;
            foreach (var item in cookieList)
            {
                System.Web.HttpContext.Current.Response.Cookies.Remove(item);
                System.Web.HttpContext.Current.Request.Cookies.Remove(item);
            }

            System.Web.HttpContext.Current.Response.Cookies.Clear();
            System.Web.HttpContext.Current.Request.Cookies.Clear();
        }
```

Y finalmente el acceso a de cada tenant por cada request


```cs
        public static Tenant Current
        {
            get
            {
                return GetCurrent();
            }
        }

        private static Tenant GetCurrent()
        {
            //Objeto poblado para cuando sea necesario, para este ejemplo no hacia falta ya que las properties son
            //lazyloading... para optimiziar es necesairo no traer el objeto poblado completamente por cada GetCurrent().
            return new Tenant();
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


PS: Por si te lo preguntas y te llegas a decir: "Pero si con un token que para mi sea válido bastaba, o con uno que se genere en base a un string que uno mismo le indique... la gran gracia de este método es que se convierte en una <b>fabrica totalmente autosuficiente</b>, ya que genera de acuerdo a:<br>
KeyPeticion->TokenAutorización 

Saludos!<br>
John.