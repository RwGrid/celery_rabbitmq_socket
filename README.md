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

<h2>vhost config</h2>
the vhost must be created in the rabbitmq web ui , for the celery worker and the api to connect through it 
as seen in creating vhost image
<h2>Celery worker task</h2>
# the         process_comments_crawl.update_state(state='PROGRESS', meta={'status': {'current_state':'PROGRESS','status_update_message':update}}) updates  the metadata , which i can put whatever i want into it 
and in the api , using this line -->task_status = task.info.get('status'), i can extraact

@celery.task
def process_comments_crawl(youtube_link):

    print("url sent :" + youtube_link)
    process_comments_crawl.update_state(state='PROGRESS',
                                        meta={'status': {'current_state': 'PROGRESS', 'status_update_message': 'يتم سحب إحصائيات رابط اليوتيوب'}})
    youtube_data, error = process_youtube_episode(comments_allowed=True, url=youtube_link)

    if error == 'failed':
        return {"status": "WARNING", "message": "Youtube URL is wrong"}

    comments = youtube_data['comments']
    comments_lst = [cmnt['sentence'] for cmnt in comments]
    sentiment_result = []
    print("loading model...")
    # must download data--->  camel_data -i sentiment-analysis-all
    sa = SentimentAnalyzerManager.get_instance()  # Initialize and access the model
    
    print(" model finished loading...")

    count = 0
    total_comments_count = len(comments_lst)
    for sentence in comments_lst:
        obj = {}
        obj['sentence'] = sentence

        sentiment = sa.predict_sentence(sentence)

        count += 1
        update = "يتم الآن تحليل التعليقات" + str(count) + "/" + str(total_comments_count)
        process_comments_crawl.update_state(state='PROGRESS', meta={'status': {'current_state':'PROGRESS','status_update_message':update}})
        print("number of comments sentiment analyzed for now ->" + str(count))
        obj['sentiment'] = sentiment
        sentiment_result.append(obj)
    process_comments_crawl.update_state(state='PROGRESS',
                                        meta={'status': {'current_state': 'PROGRESS', 'status_update_message': 'انتهت عملية تحليل التعليقات,بدأ عملية ربط البيانات....'}})

    sentiment_stat = dict()
    sentiment_stat['negative_comments'] = len([sent for sent in sentiment_result if sent['sentiment'] == 'negative'])
    sentiment_stat['positive_comments'] = len([sent for sent in sentiment_result if sent['sentiment'] == 'positive'])
    sentiment_stat['neutral_comments'] = len([sent for sent in sentiment_result if sent['sentiment'] == 'neutral'])

    sentiment_sentences = pd.DataFrame(sentiment_result)
    comments_df = pd.DataFrame(comments)
    comments_with_sentiment = pd.merge(sentiment_sentences, comments_df, on='sentence', how='inner').astype(
        str)
    comments_with_sentiment['sentence'] = comments_with_sentiment['sentence'].apply(lambda x: remove_br(x))
    comments_with_sentiment = comments_with_sentiment.to_dict(
        orient='records')
    date_counts = defaultdict(int)

    for d in comments_with_sentiment:
        # convert ISO formatted date string to a datetime object
        date = datetime.datetime.fromisoformat(d['updated_at'].replace('Z', '+00:00')).date()
        timestamp = int(time.mktime(date.timetuple())) * 1000
        d['timestamp'] = timestamp
        date_counts[timestamp] += 1
    # convert dictionary to a list of tuples
    date_counts_list = [[(date), count] for date, count in date_counts.items()]
    # sort list by date
    date_counts_list.sort()

    clean_comments = clean_text(comments_lst)
    bigrams, top_bigram_series, top_bigram_tags = get_bigrams(corpus=" ".join(clean_comments), top_x=10)
    unigrams, top_unigram_series, top_unigram_tags = get_unigrams(corpus=" ".join(clean_comments), top_x=10)
    stats = dict()
    stats['عدد التعليقات'] = str(youtube_data['عدد التعليقات'])
    stats['عدد الإعجابات'] = str(youtube_data['عدد الإعجابات'])
    stats['عدد المشاهدات'] = str(youtube_data['عدد المشاهدات'])
    comments_info = dict()
    comments_info['unigram'] = unigrams
    comments_info['bigram'] = bigrams
    comments_info['top_unigram_series'] = top_unigram_series
    comments_info['top_unigram_tags'] = top_unigram_tags
    comments_info['top_bigram_series'] = top_bigram_series
    comments_info['top_bigram_tags'] = top_bigram_tags
    comments_info['stats'] = stats
    comments_info['comments_data'] = comments_with_sentiment
    comments_info['message'] = "successfully analyzed comments"
    comments_info['status'] = "success"
    comments_info['sentiment_stat'] = sentiment_stat
    comments_info = jsonable_encoder(comments_info)
    comments_info['engagement_metric'] = math.ceil(
        int(youtube_data['عدد المشاهدات']) / youtube_data['عدد المشتركين بالقناة'] * 100)

    comments_info['date_count_list'] = date_counts_list

    first_comment_timestamp = date_counts_list[0][0] / 1000  # dividing by 1000 is critical

    dt_object = datetime.datetime.fromtimestamp(first_comment_timestamp)

    # Subtract one day
    one_day = datetime.timedelta(days=1)
    new_dt_object = dt_object - one_day

    # Convert the datetime object back to a timestamp
    comments_info['date_count_first_date'] = int(new_dt_object.timestamp()) * 1000
    process_comments_crawl.update_state(state='SUCCESS',
                                        meta={'status': {'current_state': 'SUCCESS', 'status_update_message': 'النتيجة'}})
    return comments_info
<h2>the api receiving websocket from react, while contacting celery task repeatedly using the task id to check the 'progress' and 'success' of task</h2>

@app.websocket("/YoutubeCommentAnalysis/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: str):
    await websocket.accept()
    params_message = await websocket.receive_text()
    params = ujson.loads(params_message)
    comments_url = params.get("comments_url")

    # Start the Celery task
    task = celery_connection.send_task("app.worker.process_comments_crawl", args=[comments_url])
    result_data = None
    task_id = task.task_id
    while result_data == None:
        task = celery_connection.AsyncResult(task_id)
        if task.info is not None:
            task_status = task.info.get('status')
            if task_status is not None:
                # `task_status` is not None
                if 'current_state' in task_status:
                    current_state = task_status['current_state']
                    status_update_message = task_status['status_update_message']
                    if current_state == "SUCCESS":
                        result_data = task.result
                        print(result_data)
                        result = dict()
                        result['current_state'] = 'FINISHED'
                        result['data'] = result_data
                        await websocket.send_json(result)
                        break
                    elif current_state == 'FAILURE':
                        error_message = "Task encountered an error."
                        await websocket.send(error_message)
                        break
                    elif current_state == 'PROGRESS':
                        await websocket.send_json(task_status)
                else:
                    if task.state == 'SUCCESS':
                        logging.info("task is too small to get progress, finished fast")
                        print("task is too small to get progress, finished fast")
                        result_data = task.result
                        result = dict()
                        result['current_state'] = 'FINISHED'
                        result['data'] = result_data
                        await websocket.send_json(result)
                        break
            else:
                # 'status' is not found in `task.info`
                print("task_status is not found in task.info")
        else:
            # `task.info` is None
            print("task.info is None, celery worker not online")
            logging.error("task.info is None, celery worker not online")

        await asyncio.sleep(0.2)
and the react bit
<h2>react bit socket send params using socket  to api  and client id to distinguish more than one client</h2>
 useEffect(() => {
    if (dataQueried !== null) {
      const urlObj = {};
      urlObj.youtube_url = dataQueried;
      setDataExists(false);
      setLoading(true);

      const clientID = `${currentUserName}___${getCurrentDateTime()}`;
      const ws = new WebSocket(
        `wss://`
          .concat(
            `${process.env.REACT_APP_BACKEND_API_WEBSOCKET_IP}:${process.env.REACT_APP_BACKEND_API_WEBSOCKET_PORT}`
          )
          .concat(`/YoutubeCommentAnalysis/`.concat(clientID))
      );
      ws.onopen = () => {
        console.log("WebSocket connection opened.");
        const params = {
          comments_url: dataQueried,
          // Add more parameters as needed
        };

        // Send the parameters as the initial message
        ws.send(JSON.stringify(params));
      };

      ws.onmessage = (event) => {
        const data = JSON.parse(event.data);

        // Handle data received from the server, which can be updates or results
        if (data.current_state === "PROGRESS") {
          // Update the state with progress data
          setProgressUpdate(data.status_update_message);
        } else if (data.current_state === "FINISHED") {
          // Handle the final result
          // Update the state or perform any necessary actions
          showCrawlingYoutubeResults(data.data);
        }
      };

      ws.onclose = () => {
        console.log("WebSocket connection closed.");
      };

      setSocket(ws);

      // Cleanup the WebSocket connection when the component unmounts
      return () => {
        ws.close();
        setProgressUpdate("");
      };
    }
  }, [dataQueried, mutate]);
