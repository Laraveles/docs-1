# Laravel Horizon

- [Introducción](#introduction)
- [Instalación](#installation) 
    - [Configuración](#configuration)
    - [Autenticación del Dashboard](#dashboard-authentication)
- [Ejecutar Horizon](#running-horizon) 
    - [Desplegar Horizon](#deploying-horizon)
- [Etiquetas](#tags)
- [Notificaciones](#notifications)
- [Métricas](#metrics)

<a name="introduction"></a>

## Introducción

Horizon proporciona un hermoso *dashboard* y una configuración controlada por código para sus colas de Redis gestionadas por Laravel. Horizon permite monitorear de forma sencilla las métricas clave de su sistema de colas (*queues*), tales como el rendimiento de trabajos (*jobs*), el tiempo de ejecución y las fallas del trabajos.

Toda la configuración se almacena en un único y sencillo archivo de configuración, permitiendo que permanezca en el lugar de control del código donde todo su equipo puede colaborar.

<a name="installation"></a>

## Instalación

> {note} Debido a su uso de señales de proceso asincrónicas, Horizon requiere PHP 7.1+.

Puede usar Composer para instalar su proyecto Laravel:

    composer require laravel/horizon
    

Después de instalar Horizon, publique sus recursos (*assets*) usando el comando Artisan `vendor:publish`:

    php artisan vendor:publish --provider="Laravel\Horizon\HorizonServiceProvider"
    

<a name="configuration"></a>

### Configuración

Después de publicar los *assets* de Horizon, su fichero de configuración principal se ubicará en `config/horizon.php`. Este fichero de configuración permite configurar las opciones de sus *workers* y cada opción incluye una descripción de su propósito, por lo tanto, asegúrese de explorar este fichero completamente.

#### Opciones de balance

Horizon le permite elegir entre tres estrategias de balance: `simple`, `auto`, y `false`. La estrategia `simple`, la cual es la predeterminada, divide los trabajos entrantes uniformemente entre los procesos:

    'balance' => 'simple',
    

La estrategia `auto` ajusta el número de procesos de trabajo por cola basada en la actual carga de trabajo de la cola. Por ejemplo, si su cola de `notifications` tiene 1,000 trabajos en espera mientras su cola `render` está vacía, Horizon distribuirá más trabajos a su cola de `notifications` hasta que esté vacía. Cuando la opción `balance` se establece a `false`, se utilizará el funcionamiento predeterminado de Laravel, el cual procesa colas en el orden que se estableció en la configuración.

<a name="dashboard-authentication"></a>

### Autenticación del Dashboard

Horizon expone un *dashboard* en `/horizon`. Por defecto, solo podrá acceder a este *dashboard* en el entorno `local`. Para definir una política de acceso más específica para el *dashboard*, deberá usar el método `Horizon::auth`. El método `auth` acepta un *Callback* la cual deberá retornar `true` o `false`, indicando si el usuario debería tener acceso al *dashboard* de Horizon:

    Horizon::auth(function ($request) {
        // return true / false;
    });
    

<a name="running-horizon"></a>

## Ejecutar Horizon

Una vez que haya configurado sus *workers* en el fichero de configuración `config/horizon.php`, puede iniciar Horizon usando el comando Artisan `horizon`. Este único comando iniciará todos los *workers* configurados:

    php artisan horizon
    

Puede pausar el proceso de Horizon e instruirlo para continuar procesando trabajos utilizando `horizon:pause` y los comandos Artisan `horizon:continue`:

    php artisan horizon:pause
    
    php artisan horizon:continue
    

Puede finalizar el proceso principal de Horizon utilizando el comando Artisan `horizon:terminate`. Todos los trabajos que Horizon esté ejecutando en ese momento se completarán y a continuación se detendrá su ejecución:

    php artisan horizon:terminate
    

<a name="deploying-horizon"></a>

### Desplegar Horizon

Si está desplegando Horizon en un servidor online, debe configurar un monitor de procesos para monitorear para monitorear el comando `php artisan horizon` y re-ejecutarlo si se detiene de forma inesperada. Al desplegar código nuevo a su servidor, tendrá que detener el proceso principal de Horizon para que pueda ser reiniciado por su monitor de procesos y recibir los cambios de código.

Puede finalizar el proceso principal de Horizon utilizando el comando Artisan `horizon:terminate`. Todos los trabajos que Horizon esté ejecutando en ese momento se completarán y a continuación se detendrá su ejecución:

    php artisan horizon:terminate
    

#### Configuración de Supervisor

Si está utilizando el monitor de procesos *Supervisor* para gestionar su proceso `horizon`, el siguiente fichero de configuración deberá ser suficiente:

    [program:horizon]
    process_name=%(program_name)s
    command=php /home/forge/app.com/artisan horizon
    autostart=true
    autorestart=true
    user=forge
    redirect_stderr=true
    stdout_logfile=/home/forge/app.com/horizon.log
    

> {tip} Si no se encuentra cómodo gestionando sus propios servidores, considere utilizar [Laravel Forge](https://forge.laravel.com). Forge provee servidores PHP 7+ con todo lo que necesita para ejecutar aplicaciones modernas y robustas de Laravel con Horizon.

<a name="tags"></a>

## Etiquetas

Horizon permite asignar "etiquetas" a trabajos, incluyendo *mailables*, *event broadcasting*, notificaciones y colas de *event listeners*. De hecho, Horizon etiquetará de forma inteligente y automática la mayoría de los trabajos dependiendo de los modelos Eloquent que estén relacionados con el trabajo. Por ejemplo, revise el siguiente *job*:

    <?php
    
    namespace App\Jobs;
    
    use App\Video;
    use Illuminate\Bus\Queueable;
    use Illuminate\Queue\SerializesModels;
    use Illuminate\Queue\InteractsWithQueue;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Foundation\Bus\Dispatchable;
    
    class RenderVideo implements ShouldQueue
    {
        use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
        /**
         * The video instance.
         *
         * @var \App\Video
         */
        public $video;
    
        /**
         * Create a new job instance.
         *
         * @param  \App\Video  $video
         * @return void
         */
        public function __construct(Video $video)
        {
            $this->video = $video;
        }
    
        /**
         * Execute the job.
         *
         * @return void
         */
        public function handle()
        {
            //
        }
    }
    

Si este trabajo se encuentra en una cola con una instancia `App\Video`, por ejemplo, que tiene un `id` con valor `1`, automáticamente recibirá la etiqueta `App\Video:1`. Esto se debe a que Horizon examina las propiedades del trabajo buscando modelos Eloquent. Si se encuentran modelos Eloquent, Horizon inteligentemente etiquetará el trabajo utilizando el nombre de la clase y la clave principal del modelo:

    $video = App\Video::find(1);
    
    App\Jobs\RenderVideo::dispatch($video);
    

#### Etiquetado manual

Si prefiere definir las etiquetas de forma manual, puede definir un método `tags` en la clase:

    class RenderVideo implements ShouldQueue
    {
        /**
         * Get the tags that should be assigned to the job.
         *
         * @return array
         */
        public function tags()
        {
            return ['render', 'video:'.$this->video->id];
        }
    }
    

<a name="notifications"></a>

## Notificaciones

> **Note:** Antes de utilizar las notificaciones, deberá agregar el paquete Composer `guzzlehttp/guzzle` a su proyecto. Cuando configure Horizon a enviar notificaciones SMS, también deberá revisar los [requerimientos para el driver de notificación Nexmo](https://laravel.com/docs/5.5/notifications#sms-notifications).

Si prefiere recibir notificaciones cuando una de las colas tenga un tiempo de espera muy largo, se pueden utilizar los métodos `Horizon::routeMailNotificationsTo`, `Horizon::routeSlackNotificationsTo` y `Horizon::routeSmsNotificationsTo`. Podrá ejecutar estos métodos desde el `AppServiceProvider` de su aplicación:

    Horizon::routeMailNotificationsTo('example@example.com');
    Horizon::routeSlackNotificationsTo('slack-webhook-url', '#channel');
    Horizon::routeSmsNotificationsTo('15556667777');
    

#### Configurar umbrales de tiempo de espera de la notificación

Podrá configurar cuántos segundos se consideran una "larga espera" dentro del fichero de configuración `config/horizon.php`. La opción de configuración `waits` dentro de este fichero, permite controlar el umbral de espera por cada combinación conexión/cola:

    'waits' => [
        'redis:default' => 60,
    ],
    

<a name="metrics"></a>

## Métricas

Horizon incluye un *dashboard* de métricas el cual proporciona información sobre sus trabajos y tiempos de espera de la colas y rendimiento. Con el fin de añadir datos a este *dashboard*, deberá configurar el comando de Artisan `snapshot` para ejecutarse cada cinco minutos a través del [scheduler](/docs/{{version}}/scheduling) de su aplicación:

    /**
     * Define the application's command schedule.
     *
     * @param  \Illuminate\Console\Scheduling\Schedule  $schedule
     * @return void
     */
    protected function schedule(Schedule $schedule)
    {
        $schedule->command('horizon:snapshot')->everyFiveMinutes();
    }