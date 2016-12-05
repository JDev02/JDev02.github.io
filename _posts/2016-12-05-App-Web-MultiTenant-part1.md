---
layout: post
title: Aplicación Web Stateless MultiTetant Parte 1
subtitle: MultiTetant, Stateless. Sin Session, y datos mínimos del tenant y autorización (Empresa, User, etc)
---

Hola, hoy mostraré como crear una suerte de Tenant Factory para aplicaciones web multi tenant (en realidad es un wrapper).<br>
En esta oportunidad la aplicación afectada tenia un Front End bastante pulido, pero a pesar de eso, el back respondía de forma casi sincrónica, por lo cual se tuvo que refactorizar.

En mi experiencia desarrollando aplicaciones web, muchas veces me he encontrado con aplicaciones multi tenant (aka multi empesa, multi inquilino...) donde la mayoría de los proyectos con los que he tenido que trabajar están hechos básicamente apoyándose de la Session (System.Web) con la cual manejan lo datos de Session de cada usuario, guardando como mínimo estos dos datos:<br>
1.-Usuario<br>
2.-Empresa
<br>
Otros más osados guardan directamente connectionString...<br>


En primer lugar, ya he hablado sobre lo mal que esta usar Session en aplicaciones web, y lo que significa mantener datos perssitentes de configuración/navegación en la memoria de la aplicación (por ejemplo ante un reinicio de la app, todos los usuarios perderían lo que están haciendo en ese momento... su Session ya no sería válida para después del reinicio... se podría usar otro provider de session, pero aun asi no lo recomiendo.<br> Las peticiones a la Session se bloquean (es un objeto "seguro"), por lo cual no consigues mucho creando un front-end asíncrono y elegante, si tu back es sincrono (una suerte de fifo)).

La idea es que ésta misma solución la puedan llevar a una dll distinta a la de la aplicación, para luego poder compartir con distintas Areas u otras webs de su misma propiedad, también sería apropiado aplicar algo de SOLID, para convertirla en una interfaz, y mejorar algunos aspectos para después si se requiere poder inyectar. Pero vamos, para el ejemplo se usará un método estático que es el encargado de crear/validar.

Los datos serán guardados en cookies, ya que el front usaba peticiones Ajax, sin configurar un método/token de autentificación (ni hablar de cabeceras o jwt)

En la aplicación que tocó refactorizar existián básicamente dos propiedades que eran utilizadas para identificar la empresa donde estaba logeado el usuario actual, una era el usuario, y otra para la empresa (Anteriormente se guardaba en Session["Usuario"], Session["Empresa"]....)<br>
En la solucion Tenant que estamos viendo estas propiedades disponen de Get, el Set es privado, y en realidad no existe, ya que la única forma de crearlos es a través del método Tenant.Create({...})

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


Luego creamos un constructor privado para evitar que alguien en un lugar equivocado logee más veces a un usuario, asi solo le delegamos la responsabilidad al metodo static Create:


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

Y finalmente siguiendo el estilo de .NET (Current property) el acceso a de cada tenant por cada request:


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


Y para utilizarla podemos crear un método de login similar a este:

```cs
        [HttpPost]
        public ActionResult Login(LoginModel model, string returnUrl)
        {
            if (ModelState.IsValid)
            {
                try
                {
                    //process login...

                    //Login().IsValid...

                    Tenant.Create(model.UserName, model.CustomerName);
                    //redirect to home page...
                }
                catch (Exception e)
                {
                    ModelState.AddModelError("", e.Message);
                    return View(model);
                }
                return RedirectToLocal(returnUrl);
            }

            // If we got this far, something failed, redisplay form
            ModelState.AddModelError("", "The user name or password provided is incorrect.");
            return View(model);
        }


        [HttpPost]
        public ActionResult LogOff()
        {
            Tenant.Current.LogOut();
            return RedirectToAction("Login", "Account");
        }
```



Demás está decir que la pueden adaptar a su necesidad e incluirle propiedades, o funcionalidad extra... y que si bien ya existen soluciones similares (de forma static) esta es custom, y dejaré el codigo en mi github para que la puedan bajar, modificar y utilizar. <br>En la próxima parte veremos como agregar una Session/Cache a cada Tenant en especial
generando una propiedad similar a esta:



```cs
        public ICache Cache
        {
            get { return {....} }
        }
```


Antes de terminar, debo mencionar que con esto no consiguen un backend asyn, pero definitivamente lo mejoran. <br>Ahora si a sus peticiones/métodos/Actions que utilicen alguno de los datos del tenant, los crean con Async Result, ahí lograran apreciar el cambio.<br> Otra cosa no menos importante, es que toda esta solución aplica para aplicaciones con mucha concurrencia, donde les sugiero trabajen orientado a datos y separen la V (de la estructura típica que ofrece MVC .NET) en otra subcapa MVW (model view whatever; esta corre en el navegador)

Espero les sirva,

Saludos!


Saludos!<br>
John.