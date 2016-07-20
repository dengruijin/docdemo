
***************
VDSM源码分析
***************

启动分析
=========

vdsm的启动文件是/usr/share/vdsm/vdsm，即vdsm-ovirt-3.5/vdsm/vdsm
该文件中首先调用run()函数，一切从此开始：

.. code-block:: python

  if __name__ == '__main__':
    __assertVdsmUser()
    __assertLogPermission()
    __assertSudoerPermissions()

    if not config.getboolean('vars', 'core_dump_enable'):
        resource.setrlimit(resource.RLIMIT_CORE, (0, 0))

    if os.getsid(0) != os.getpid():
        # Modern init systems such as systemd can provide us a clean daemon
        # environment. So we setpgrp only when run by traditional init system
        # that does not prepare the environment.
        os.setpgrp()
    argDict = parse_args()
    run(**argDict)

run函数的关键调用流程如下：
::

  run()
    |-serve_clients(log)
    |-cif = clientIF.getInstance(irs, log)
    |                   |-cls._instance = clientIF(irs, log, scheduler)
    |                          |-self._createAcceptor(host, port)
    |                          |-self._prepareJSONRPCBinding()
    |                          |-json_binding = BindingJsonRpc(...)
    |                          |    |-self._executor = executor.Executor()
    |                          |    |-self._server = JsonRpcServer(bridge, cif, self._executor.dispatch)
    |                          |-self.bindings['jsonrpc'] = json_binding
    |                          |-stomp_detector = StompDetector(json_binding)
    |                          |-self._acceptor.add_detector(stomp_detector)
    |-cif.start()
        |-binding.start()
        |   |-self._executor.start()
        |-self.thread = threading.Thread(target=self._acceptor.serve_forever,name='Detector thread')
        |-self.thread.setDaemon(True)
        |-self.thread.start()
其中JsonRpcServer是位于yajsonrpc库中的JsonRpc服务器类，负责RPC服务端的底层实现，
而vdsm中的BindingJsonRpc是JsonRpc服务的发布者，vdsm守护进程启动时将调用BindingJsonRpc
的start()方法来启动JsonRpc服务：

file: vdsm-4.16/vdsm/rpc/BindingJsonRpc.py

.. code-block:: Python

  # 实例化BindingJsonRpc
  def __init__(self, bridge, scheduler, cif):
      # workers_count 是JSONRPC开启的最大线程(worker)数目，默认为8
      # max_tasks 是最大支持的task数目，默认每个线程最多支持10个task, 所以max_tasks=80
      self._executor = executor.Executor(name="jsonrpc.Executor",
                                         workers_count=_THREADS,
                                         max_tasks=_TASKS,
                                         scheduler=scheduler)
      # dispatch是Executor中的一个函数
      self._server = JsonRpcServer(bridge, cif, self._executor.dispatch)
      self._reactors = []

  # .... other codes .....
  # ......................
  # 启动BindingJsonRpc服务
  def start(self):
      self._executor.start() #该函数会创建所有线程(worker)

      # serve_requests是JsonRpcServer中的函数，它不同地从JsonRpcServer的队列中取请求进行处理
      t = threading.Thread(target=self._server.serve_requests,
                           name='JsonRpcServer')
      t.setDaemon(True)
      t.start()

file: vdsm-4.16/lib/yajsonrpc/__init__.py:

.. code-block::Python

  # JsonRpcServer实例化是的动作：
  def __init__(self, bridge, cif, threadFactory=None):
      self._bridge = bridge
      self._cif = cif
      self._workQueue = Queue()  # Queue是python内置类
      self._threadFactory = threadFactory
