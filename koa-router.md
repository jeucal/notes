# koa-router

有两个文件: layer.js 和 router.js.  
例如：route.get('/user/:id', function f1(ctx, next) {})  
route是一个router对象，会生成一个layer对象，存储具体的路由信息  
每次调用router的get, use等方法，都是新生成了一个layer对象，放在route.stack数组里

基本原理：
1. 一个router对象主要有两个属性：stack[]，params{}；  
2. stack是一个数组，元素是layer对象，每当定义一个路由的时候，事实上就是添加了个一个layer对象到stakck中；  
3. 一个layer对象代表了一条路由信息，一个layer对象主要包括：一个path，一个method数组，可以为0个或者多个元素，如：[], ['GET'], ['GET', 'POST']，以及一个数组stack，包含了middleware。可以把一个layer理解为一个层，每个router包含了多个层，当有request进来时，router会遍历每个layer层，根据ctx.path和ctx.method判断该层是否被调用；  
4. params结构为：{参数:处理函数, ...}, 例如{'id': function f1(id, ctx, next) {}, ...}, 每当调用如：route.param('id', function f1(id, ctx, next) {})时，首先将id和f1放入到route.params中，然后遍历route.stack, 调用每个layer的param方法, 判断当前layer的path中是否有id参数，如果有则将f1插入到这个layer的stack的开头。作用就是：可以把对该path的参数处理，单独出来，放到其他middleware的开头。另外，每当给route添加一个layer层时，会遍历params，看哪一个param在新增的layer的path中，给该layer的stack添加param处理函数。最终实现的效果就是：一个router.stack上所有的layer层，都会有router.params指定的参数处理函数；  
5. 当使用router.use(middleware)方法时，layer的path没有指定，这相当于对经过该路由的所有request都使用该middleware，例如：router.use(session());  
6. 当使用router.use('/path', middleware)时，指定了path，但没有指定method，这相当于对该path的所有request都调用middleware；  

## layer
  #### 一个layer对象有以下属性：  
  * this.path: 路由路径 '/user/:id'  
  * this.method: 路由方法 'GET'  
  * this.stack: 对应的中间件数组 [f1],可以是多个[f1, f2]. 如果是有一个中间件，比如本例的f1，会转化为数组[f1]
  * this.regexp: 把this.path转化成的正则对象
  * this.paramNames: this.path的参数列表：[{name: 'id', ...}]
  * this.name: 别名

  #### layer.param方法
  使用方法：layer1.param('id', function f1(id, ctx, next) {// do something about id}); 该函数会把f1插入到layer1.stack的开头。作用就是提前处理该路径路由的参数。
  

## router
var router = require('koa-router')();  
router.stack 是一个array, 里面每个元素是一个layer对象

#### route.use
  use方法有三种使用方法：
  1. route.use('/path', (ctx, next) => {})
  这会调用route.register(path, [], middleware)，生成一个新的layer放入route.stack中。注意第二个参数method为空数组[].  
  关键点：这种调用方法，没有指定method, 所以实现的效果就是：只要符合path的request，都会被这个中间件处理。

  2. route.use('/path', route1.routes(), route2.routes()
  把route1和route2的stack中的layer都提取出来，放入route的stack中.
  关键点：route1和route2中的layer，要先加上route的prefix, 还有params，再放到route中。最终实现的效果就是：route集合了route1和route2定义的所有路由，就好像直接再route上定义路由一样。

  3. route.use(middleware) 
  没有指定path，也没有method，所有request都会被middleware处理，例如：route.use(session())

  #### route.routes
  调用该方法返回一个函数dispatch(ctx, next)，该函数就是一个中间件，内部将这个route的layer都给串联起来，然后compose，并返回。因此可以这样使用：app.use(route.routes()) => app.use(dispatch)。
  该函数也可以供route.use调用，因为该方法在返回的dispatch上定义了：dispatch.router = this,当r1.use('/path', r2.routes())的时候, r1.use会判断出dispatch上有一个router，因此就可以把r2的stack里面的所有layer加入到r1当中. 


