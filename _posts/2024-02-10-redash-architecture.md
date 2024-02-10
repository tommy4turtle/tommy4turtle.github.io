---
layout: post
title: Redash 아키텍처 구조 파악
tags: Redash, rq_scheduler, 리대시, 대시보드, 데이터 분석
math: true
date: 2024-02-10 15:00 +0900
toc:  true
author: Tommy Kim
categories: 오픈소스_분석
---
오픈소스 시각화 툴인 Redash 구조에 대해 알아보도록 하겠습니다.
## Redash란 무엇인가
Redash helps you make sense of your data
{: .message }


Redash는 쿼리를 이용하여 간단하게 대시보드를 구축할 수 있는 오픈소스입니다.
사용자가 SQL, Python 등 다양한 데이터 소스에서 직접 쿼리를 실행하고, 결과를 동적 대시보드와 그래프로 시각화할 수 있게 해줍니다.
### 다양한 소스
Redash는 PostgreSQL, MySQL, Google BigQuery, MongoDB, Presto, Google Sheets, AWS Athena 등 다양한 데이터 소스를 지원합니다.
### 대시보드
시각화 형식(차트, 코호트 분석, 피벗 테이블, 박스플롯, 지도, 카운터, 샌키 다이어그램, 선버스트 차트, 워드 클라우드 등)을 활용하여 데이터 결과를 시각화하고, 여러 데이터 소스로부터 정보를 수집하여 테마별 대시보드를 구성하는 것을 포함합니다. 
### 스케줄러
Redash의 스케쥴러는 사용자가 정의한 쿼리를 자동으로 실행하게 해주는 기능입니다. 이를 통해 데이터를 주기적으로 새로고침하고, 대시보드를 최신 상태로 유지할 수 있습니다. 사용자는 쿼리를 작성한 후, 해당 쿼리의 설정에서 실행 빈도(예: 매시간, 매일, 매주 등)를 지정할 수 있습니다.
- 조건 설정: 쿼리를 실행한 후, 결과가 특정 조건(예: 결과값이 특정 수치를 초과하거나 미만일 때)을 만족하는지 여부를 검사하는 조건을 설정합니다.
- 알림 채널 추가: 경고를 받을 방법을 선택합니다. Redash는 이메일, Slack, 웹훅(Webhook) 등 다양한 알림 채널을 지원합니다.
- 경고 활성화: 설정한 조건과 알림 채널을 저장하면, 조건이 만족될 때마다 알림을 받게 됩니다.

[Redash 공식페이지](https://redash.io/)
## 전체 구조
<div class="mermaid"> 
  graph LR
    subgraph Redash Server
    A[Flask Application] -->|API Requests| D[Data Sources]
    A -->|Manage| B[RQ Workers]
    A -->|Schedule| C[RQ Scheduler]
    B -->|Execute Queries| D
    C -->|Periodic Tasks| B
    end

    subgraph External
    E[Web Browser] -->|User Interaction| A
    F[External Applications] -->|API Calls| A
    end
</div>
- 프론트엔드 : React
- 백엔드: Python (Flask 라이브러리)
- DB : PostgreSQL
- Job Queue : Redis와 RQ (Redis Queue)

## 살펴볼 기능
전체 소스 코드 중 궁금한 부분을 위주로 소스 코드를 확인해보겠습니다. 확인할 부분은 3가지 영역입니다.
1. 쿼리 등록 후 쿼리 실행
2. 스케줄러 동작 방식

### 어떻게 쿼리가 실행될까?

{% highlight python %}
def enqueue_query(query, data_source, ...):
  ...
  queue = Queue(queue_name) # from rq import Queue
  enqueue_kwargs = {
      "user_id": user_id,
      "scheduled_query_id": scheduled_query_id,
      "is_api_key": is_api_key,
      "job_timeout": time_limit,
      "failure_ttl": settings.JOB_DEFAULT_FAILURE_TTL,
      "meta": {
          "data_source_id": data_source.id,
          "org_id": data_source.org_id,
          "scheduled": scheduled_query_id is not None,
          "query_id": metadata.get("query_id"),
          "user_id": user_id,
      },
  }
  ... 
  job = queue.enqueue(execute_query, query, data_source.id, metadata, **enqueue_kwargs)
  ...
    return job
{% endhighlight %}
1.rq 라이브러리의 Queue을 활용하여 execute_query 함수, query 문자열, 데이터 소스, 메타데이터 등 큐에 enqune 시킵니다.
{% highlight python %}
def execute_query(query, data_source_id, ...):
    try:
        return QueryExecutor(
            query,
            data_source_id,
            user_id,
            is_api_key,
            metadata,
            scheduled_query_id is not None,
        ).run()
    except QueryExecutionError as e:
        models.db.session.rollback()
        return e
{% endhighlight %}
2.execute_query함수는 QueryExecutor.run()을 활용하여 쿼리 실행
{% highlight python %}
class QueryExecutor(object):

    def run(self):
        signal.signal(signal.SIGINT, signal_handler)
        started_at = time.time()

        logger.debug("Executing query:\n%s", self.query)
        self._log_progress("executing_query")

        query_runner = self.data_source.query_runner
        annotated_query = self._annotate_query(query_runner)

        try:
            data, error = query_runner.run_query(annotated_query, self.user)
...
{% endhighlight %}
3.QueryExecutor는 각 data_source의 query_runner를 활용하여 쿼리를 실행시켜 실제 쿼리 결과를 얻습니다.

{% highlight python %}

def run_query(self, query, user):
      ev = threading.Event()
      thread_id = ""
      r = Result()
      t = None

      try:
          connection = self._connection()
          thread_id = connection.thread_id()
          t = threading.Thread(target=self._run_query, args=(query, user, connection, r, ev))
          t.start()
          while not ev.wait(1):
              pass
      except (KeyboardInterrupt, InterruptException, JobTimeoutException):
          self._cancel(thread_id)
          t.join()
          raise

      return r.json_data, r.error

def _run_query(self, query, user, connection, r, ev):
      try:
          cursor = connection.cursor()
          logger.debug("MySQL running query: %s", query)
          cursor.execute(query)

          data = cursor.fetchall()
          desc = cursor.description

          while cursor.nextset():
              if cursor.description is not None:
                  data = cursor.fetchall()
                  desc = cursor.description

          # TODO - very similar to pg.py
          if desc is not None:
              columns = self.fetch_columns([(i[0], types_map.get(i[1], None)) for i in desc])
              rows = [dict(zip((column["name"] for column in columns), row)) for row in data]

              data = {"columns": columns, "rows": rows}
              r.json_data = json_dumps(data)
              r.error = None
          else:
              r.json_data = None
              r.error = "No data was returned."

          cursor.close()
      except MySQLdb.Error as e:
          if cursor:
              cursor.close()
          r.json_data = None
          r.error = e.args[1]
      finally:
          ev.set()
          if connection:
              connection.close()

{% endhighlight %}
4.가장 보편적인 데이터 소스인 mysql query_runner 입니다.

### 스케쥴러 동작 방식

{% highlight python %}

def refresh_queries():
    started_at = time.time()
    logger.info("Refreshing queries...")
    enqueued = []
    for query in models.Query.outdated_queries():
        if not _should_refresh_query(query):
            continue

        try:
            query_text = _apply_default_parameters(query)
            query_text = _apply_auto_limit(query_text, query)
            enqueue_query(
                query_text,
                query.data_source,
                query.user_id,
                scheduled_query=query,
                metadata={"query_id": query.id, "Username": query.user.get_actual_user()},
            )
            enqueued.append(query)
        except Exception as e:
            message = "Could not enqueue query %d due to %s" % (query.id, repr(e))
            logging.info(message)
            error = RefreshQueriesError(message).with_traceback(e.__traceback__)
            sentry.capture_exception(error)

    status = {
        "started_at": started_at,
        "outdated_queries_count": len(enqueued),
        "last_refresh_at": time.time(),
        "query_ids": json_dumps([q.id for q in enqueued]),
    }

    redis_connection.hset("redash:status", mapping=status)
    logger.info("Done refreshing queries: %s" % status)

{% endhighlight %}
rq_scheduler를 사용하여, 실행시켜야 할 쿼리 리스트를 30초마다 Fetch하여 동작시킵니다.

## 마치는 말
실제 핵심 동작 코드 외에도 동시성 관리, 권한 제어 등 많은 소스 코드가 있었지만, 가장 중요하다고 생각하는 코드 위주로 살펴보았습니다. 앞으로도 여러 오픈소스 코드를 분석하며 어떤 식으로 동작하는 지 살펴보면 데이터 엔지니어링 공부에 큰 도움이 될 꺼 같습니다.