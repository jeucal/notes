# generic-pool & pg-pool & pg
generic-pool & pg-pool: 提供pg的连接池管理  
pg: postgresql的驱动

## generic-pool 通用的连接池管理 [github](https://github.com/coopernurse/node-pool)
  #### 提供了三个基本方法： 
  - pool.acquire: 获取一个连接
  - pool.release: 将一个连接释放，放回到pool中
  - pool.destroy: 将一个连接销毁

  #### 创建一个pool：genericPool.createPool(factory, opts)
  - factory: 被管理的对象。generic pool只是提供一个管理pool的工具，具体管理什么东西，比如postgresql，还是其他东西，由factory提供. factory提供一下两个函数：
    - create: 创建一个连接，比如创建一个到postgresql的连接
    - destroy: 销毁一个连接
  - opts: 参数
    - max, min:  池子的容量
    - idleTimeoutMillis: 当一个连接空闲多长时间后关闭。pool会保留至少min个连接

## pg-pool postgresql的连接池 [github](https://github.com/brianc/node-pg-pool)
  #### 该模块把generic-pool包装了一下: 把pg.client当作factory，传入generic-pool。主要方法：
  - connect: 返回一个连接，其实就是调用generic-pool的acquire获取一个连接client（pg.client对象），然后就可以在client对象上进行query等操作。最后需要自己client.release，把client还给pool.
    - 需要注意：connect方法从generic-pool里面acquire了一个client后，在这个client上绑定了一个release方法
    - 这个release方法，接受一个参数err，如果err为空，则调用generic-pool的release，意味着只是将连接释放
    - 如果这个err参数存在，则调用generic-pool的destroy，将client直接销毁
  - query: 调用connect获取一个client，进行query操作，最后自动调用client.release

## pg postgresql的驱动 [github](https://github.com/brianc/node-postgres)
  #### pg模块是底层的postgresql数据库驱动，可以创建连接到pg数据库的连接。如果加上上面两个模块，则可以创建一个连接池。  
  - pg.Client: var client = new pg.Client(opts): 初始化一个到postgresql数据库的连接, opts是连接数据库的参数
    - client.connect: 创建一个连接，调用后pg会创建 
    - client.end: 销毁一个连接
    - client.query: 添加一个query到query queue, 
    上面就是基本的连接postgresql数据库的用法
  - pg.Pool: 一个pg-pool对象: var pool = new pg.Pool(opts)，参见上面的pg-pool
  - pg.Query: query对象，由pg.Client返回

