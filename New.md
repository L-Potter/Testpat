apt-get install gnome-keyring  
apt-get install libsecret-1-0  
apt-get install libsecret-1-dev  
ps aux | grep gnome  
apt install libsecret-tools   
secret-tool --version  
secret-tool store --label="TestKey" service test username testuser  
* Init DEfault keyring password for gui encoding  
secret-tool lookup service test username testuser  
sudo make --directory=/usr/share/doc/git/contrib/credential/libsecret/  
git config --global credential.helper /usr/share/doc/git/contrib/credential/libsecret/git-credential-libsecret  
git config --global user.name "danny"  
git config --global user.email "danny@..."  

git add  
git commit -m   
git push origin main  
#gui  
sudo apt install seahorse  
secret-tool search server github.com  

```python
# coding=UTF-8
from __future__ import absolute_import

import os
import pika
import pickle
import logging
import itertools
from urlparse import urlparse
from ast import literal_eval
from celery import Celery
from celery.result import AsyncResult
from celery.task.control import inspect, revoke
from . import get_celery_queues
from concurrent.futures import ThreadPoolExecutor


# set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'gocloud.settings')

from django.conf import settings  # noqa

app = Celery('test')

# Using a string here means the worker will not have to
# pickle the object when using Windows.
app.config_from_object('django.conf:settings')
app.autodiscover_tasks(lambda: settings.INSTALLED_APPS)
logger = logging.getLogger(__name__)
pool = ThreadPoolExecutor(10)


class StackTask(app.Task):
    abstract = True
    send_error_emails = True

    """
    Handlers
    after_return(self, status, retval, task_id, args, kwargs, einfo)
    Handler called after the task returns.

    Parameters:
    status - Current task state.
    retval - Task return value/exception.
    task_id - Unique id of the task.
    args - Original arguments for the task that returned.
    kwargs - Original keyword arguments for the task that returned.
    einfo - ExceptionInfo instance, containing the traceback (if any).
    The return value of this handler is ignored.

    on_failure(self, exc, task_id, args, kwargs, einfo)
    This is run by the worker when the task fails.

    Parameters:
    exc - The exception raised by the task.
    task_id - Unique id of the failed task.
    args - Original arguments for the task that failed.
    kwargs - Original keyword arguments for the task that failed.
    einfo - ExceptionInfo instance, containing the traceback.
    The return value of this handler is ignored.

    on_retry(self, exc, task_id, args, kwargs, einfo)
    This is run by the worker when the task is to be retried.

    Parameters:
    exc - The exception sent to retry().
    task_id - Unique id of the retried task.
    args - Original arguments for the retried task.
    kwargs - Original keyword arguments for the retried task.
    einfo - ExceptionInfo instance, containing the traceback.
    The return value of this handler is ignored.

    on_success(self, retval, task_id, args, kwargs)
    Run by the worker if the task executes successfully.

    Parameters:
    retval - The return value of the task.
    task_id - Unique id of the executed task.
    args - Original arguments for the executed task.
    kwargs - Original keyword arguments for the executed task.
    The return value of this handler is ignored.
    """

    def __call__(self, *args, **kwargs):
        print('TASK STARTING: {0.name}[{0.request.id}]'.format(self))
        return super(StackTask, self).__call__(*args, **kwargs)

    def after_return(self, status, *args, **kwargs):
        print('TASK RETURNED: {0.name}[{0.request.id}]'.format(self))

    def on_failure(self, exc, *args, **kwargs):
        print('TASK FAILED: {0!r}{1.name}[{1.request.id}]'.format(exc, self))

    def on_success(self, *args, **kwargs):
        print('TASK SUCCEEDED: {0.name}[{0.request.id}]'.format(self))

    def apply_async(self, args=None, kwargs=None, task_id=None,
                    producer=None, link=None, link_error=None, **options):
        queue = options.get('queue')
        if queue and not app.amqp.queues.get(queue):
            for q in get_celery_queues():
                if queue == q.name:
                    app.amqp.queues.update({queue: q})
        return super(stackTask, self).apply_async(
            args, kwargs, task_id, producer,
            link, link_error, **options)


class TaskResult(AsyncResult):

    def _get_task_meta(self):
        backlog_limit = getattr(settings, 'CELERY_BACKLOG_LIMIT', 5000)
        if self._cache is None:
            return self._maybe_set_cache(self.backend.get_task_meta(
                self.id, backlog_limit))
        return self._cache


class TaskQueue(object):
    def __init__(self, queues=[], req_type='site'):
        super(TaskQueue, self).__init__()
        assert req_type in ['site', 'server']
        self.req_type = req_type
        self.req_type_id = req_type + '_id'
        self.queues = queues

    @staticmethod
    def _get_req_type_id(obj_task):
        if 'server_id' in obj_task:
            req_type_id = 'server_id'
        else:
            req_type_id = 'site_id'
        return req_type_id, obj_task[req_type_id]

    def _get_tasks(self, *args, **kwargs):
        raise NotImplementedError

    def get_queueing_tasks(self, *args, **kwargs):
        raise NotImplementedError

    def update_priority_by_id(self, obj_id, priority, *args, **kwargs):
        raise NotImplementedError

    def revoke_tasks_by_filter(self, obj_id, task_names, *args, **kwargs):
        raise NotImplementedError


class AMQPTask(TaskQueue):
    def __init__(self, queues=[], req_type='site', timeout=10):
        super(AMQPTask, self).__init__(queues, req_type)
        self.timeout = timeout

    def __enter__(self):
        result = urlparse(settings.BROKER_URL)
        credentials = pika.PlainCredentials(username=result.username,
                                            password=result.password)
        parameters = pika.ConnectionParameters(host=result.hostname,
                                               port=result.port,
                                               credentials=credentials)
        self.connection = pika.BlockingConnection(parameters)
        self.connection.add_timeout(self.timeout, self.connection.close)
        return self

    def __exit__(self, e_type, e_value, traceback):
        self.connection.close()

    def _get_queue_message_count(self, queue, channel):
        arguments = {'x-max-priority': settings._VM_QUEUE_MAX_PRIORITY}
        if queue != getattr(settings, 'CELERY_DEFAULT_QUEUE', 'celery'):
            arguments['x-expires'] = settings._VM_QUEUE_EXPIRES
        q = channel.queue_declare(
            queue=queue,
            durable=True,
            arguments=arguments
        )

        return q.method.message_count

    def get_message_amount(self, method=None, name_only=True):
        assert method in ['max', 'min']
        q_list = []
        with self.connection.channel() as channel:
            for queue in self.queues:
                count = self._get_queue_message_count(queue, channel)
                q_list.append({"name": queue, "count": count})
        if method is None:
            return q_list
        elif method == 'max':
            q_dict = max(q_list, key=lambda x: x['count'])
        elif method == 'min':
            q_dict = min(q_list, key=lambda x: x['count'])
        return q_dict['name'] if name_only else q_dict

    def _get_tasks(self, channel):
        tasks = []
        for queue in self.queues:
            message_count = self._get_queue_message_count(queue,
                                                          channel)
            if message_count:
                for method_frame, properties, body in channel.consume(queue):
                    payload = pickle.loads(body)
                    try:
                        obj_task = pickle.loads(payload['args'][0])
                        obj = {'method_frame': method_frame,
                               'payload': payload,
                               'properties': properties,
                               'queue': queue,
                               'site_id': obj_task.site_id}
                        if hasattr(obj_task, 'server_id'):
                            obj.update({'server_id': obj_task.server_id})
                        tasks.append(obj)
                    except Exception as e:
                        logger.warning(e)
                    if method_frame.delivery_tag == message_count:
                        break
        return tasks

    def get_queueing_tasks(self):
        with self.connection.channel() as channel:
            tasks = self._get_tasks(channel)
            channel.cancel()
            queueing_tasks = list()
            for task in tasks:
                if 'check_resource' in task['payload']['task']:
                    req_type_id, obj_id = self._get_req_type_id(task)
                    queueing_tasks.append({
                        'task_id': task['payload']['id'],
                        'priority': task['properties'].priority,
                        req_type_id: obj_id,
                        'queue': task['queue']
                    })
            return queueing_tasks

    def revoke_tasks_by_filter(self, obj_id, task_names, *args, **kwargs):
        queue = None
        with self.connection.channel() as channel:
            tasks = self._get_tasks(channel)
            for task in tasks:
                if task.get(self.req_type_id) == obj_id:
                    if task['payload']['task'] in task_names:
                        queue = task['queue']
                    channel.basic_ack(task['method_frame'].delivery_tag)
            channel.cancel()
        return queue

    def update_priority_by_id(self, obj_id, priority):
        with self.connection.channel() as channel:
            tasks = self._get_tasks(channel)
            for task in tasks:
                if task.get(self.req_type_id) == obj_id:
                    channel.basic_ack(task['method_frame'].delivery_tag)
                    args = task['payload']['args']
                    obj_task = pickle.loads(args[0])
                    obj_task.priority = priority
                    task['payload']['args'] = (pickle.dumps(obj_task),) + \
                        args[1:]
                    task['properties'].priority = priority
                    channel.basic_publish(exchange=task['queue'],
                                          routing_key=task['queue'],
                                          properties=task['properties'],
                                          body=pickle.dumps(task['payload']))
            channel.cancel()

    def delete_queue(self, queue):
        with self.connection.channel() as channel:
            channel.queue_delete(queue)
            channel.exchange_delete(queue)


def get_task_ids_in_chain(result):
    task_ids = [result.task_id]
    parent = result.parent
    while parent:
        task_ids.insert(0, parent.task_id)
        parent = parent.parent
    return task_ids


class CeleryTask(TaskQueue):
    states = ['active', 'scheduled', 'reserved']

    def __init__(self, queues, req_type='site'):
        super(CeleryTask, self).__init__(queues, req_type)
        self.insp = inspect()

    @staticmethod
    def _get_args(payload):
        try:
            args = repr(payload.strip('(\\ ",\\)')
                        ).replace('\\\\', '\\').replace('\\\\\\\\', '\\\\')
            return pickle.loads(literal_eval(args))
        except Exception as e:
            logger.error('Error while get task args: %s. payload: %s' % (
                e, payload))

    def _get_tasks(self):
        objs = list()
        futures = [pool.submit(self.get_inspect_tasks, state)
                   for state in self.states]
        tasks = itertools.chain.from_iterable(future.result()
                                              for future in futures)
        for task in tasks:
            try:
                body = task.get('request', task)
                data = self._get_args(body['args'])
                if hasattr(data, 'site_id'):
                    obj = {
                        'task_name': body['name'],
                        'site_id': data.site_id,
                        'task_id': body['id'],
                        'delivery_info': body['delivery_info'],
                        'queue': body['hostname'].split('@')[0]
                    }
                    if hasattr(data, 'server_id'):
                        obj.update({'server_id': data.server_id})
                    objs.append(obj)
            except Exception as e:
                logger.warning(e)
        return objs

    def get_queueing_tasks(self):
        tasks = self._get_tasks()
        queueing_tasks = list()
        for task in tasks:
            if 'check_resource' in task['task_name']:
                req_type_id, obj_id = self._get_req_type_id(task)
                queueing_tasks.append({
                    req_type_id: obj_id,
                    'task_id': task['task_id'],
                    'priority': task['delivery_info']['priority'],
                    'queue': task['queue']})
        return queueing_tasks

    def get_inspect_tasks(self, state):
        tasks = getattr(self.insp, state)()
        if tasks is not None:
            return list(itertools.chain.from_iterable(
                [v for k, v in tasks.iteritems()
                 if k.split('@')[0] in self.queues]))
        else:
            return []

    def revoke_tasks_by_filter(self, obj_id, task_names, *args, **kwargs):
        queue = None
        tasks = self._get_tasks()
        for task in tasks:
            if task.get(self.req_type_id) == obj_id:
                if task['task_name'] in task_names:
                    queue = task['queue']
                revoke(task['task_id'], terminate=True)
                logger.debug('Revoke %s %s and task %s' %
                             (self.req_type, obj_id, task['task_id']))
        return queue

    def update_priority_by_id(self, obj_id, priority):
        tasks = self._get_tasks()
        for task in tasks:
            if task.get(self.req_type_id) == obj_id:
                result = TaskResult(task['task_id'])
                result.backend.store_result(task['task_id'],
                                            priority, 'UPDATE')


def update_priority(queues, obj_id, priority, req_type='site'):
    assert req_type in ['site', 'server']
    with AMQPTask(queues=queues, req_type=req_type) as amqp_task:
        amqp_task.update_priority_by_id(obj_id, priority)
    Task = CeleryTask(queues=queues, req_type=req_type)
    Task.update_priority_by_id(obj_id=obj_id, priority=priority)
