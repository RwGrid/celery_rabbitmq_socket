<h2>change ubuntu containing rabbit mq: </h2>
This is necessary so that rabbitmq does not crash in watch file
with RustNotify(
OSError: OS file watch limit reached. about ["/home/ali/anaconda3/pkgs/libclang-10.0.1-default_hb85057a_2/info/recipe/parent"] (Error { kind: MaxFilesWatch, paths: ["/home/ali/anaconda3/pkgs/libclang-10.0.1-default_hb85057a_2/info/recipe/parent"] }) rabbitmq
--sudo nano /etc/sysctl.conf

  vm.max_map_count=262144
  fs.inotify.max_user_watches=524288

<h2>first thing to do is to understand the celery config on both the api serverice and the celery worker</h2>
SENTIMENT_CONFIG: dict = dotenv_values("./.env.sentiment")

sentiment_server_ip = SENTIMENT_CONFIG["SENTIMENT_IP"]  # Example IP address
is_sentiment_analysis_online = is_server_online(sentiment_server_ip)
celery_connection = None
if is_sentiment_analysis_online:
    print("> The sentiment service is working")
    logging.info("> The sentiment service is working")
    celery_connection = Celery("app", broker='amqp://' + SENTIMENT_CONFIG['CELERY_RABBITMQ_USER'] + ':' +
                                             SENTIMENT_CONFIG['CELERY_RABBITMQ_PASSWORD'] + '@' + SENTIMENT_CONFIG[
                                                 'CELERY_RABBITMQ_IP'] + ':'
                                             + SENTIMENT_CONFIG['CELERY_RABBITMQ_PORT'] + '/' + SENTIMENT_CONFIG[
                                                 'CELERY_RABBITMQ_VIRTUAL_HOST'], backend='rpc://')

else:
    print("> The sentiment service is down")
    logging.info("> The sentiment service is down")

-> as seen here , 2 notes must be taken and must be set, the backend needs to be rpc , 
for communication between the celery worker  to return values to the api. without this, a normal rest request wont work between the celery worker and the api , so a return wont be invoked.
-> the celery_rabbitmq_virtual_host must be set from the 
