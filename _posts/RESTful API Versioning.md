title: RESTful API Versioning
tags:
  - API
date: 2014-12-18 14:23:06
---

关于RESTful API Versioning，有很多讨论 [Nobody Understands REST or HTTP][1], [Versioning REST Services][2], [best-practices-for-api-versioning][3], [How are REST APIs versioned?][4]， 总结如下。

为什么需要Versioning
==================

尽管设计API的时候，我们尽可能设计完美的API，尽可能的避免修改API。但是随着业务需求的变更，API接口的变化几乎是无法避免的。

当业务需求变更的时候，可以这样选择：
1. 保持接口的兼容性。这是一种方式，但是并非能切实做到的，为了兼容必定会损失一些新特性，或者牺牲良好的代码为代价。
2. 修改API接口的同时，修改客户端。除非自己维护少量的客户端，否则这几乎是不现实的。
3. 保留旧的接口，通过版本来实现API接口的变更。

通常情况下，我们会选择第三种方式来实现API接口的变更。

<!--more-->

实现策略
=======
1. URI
    - `https://api-v1.example.com/places`
    - `https://api.example.com/v1/places`

    上述两种方式都是分别通过Path和Hostname来进行versioning。这种方式的好处直观，友好，易于理解，“复制&粘贴”更为友好；但是RESTful本身就不是“复制&粘贴”友好的。[RESTafarian](http://mikeschinkel.com/blog/whatisarestafarian/)们根本不认同这是RESTful API，因为它破坏了HATEOAS，直接称它为TUK(The URL is King)。

2. 请求体，Query参数
        POST/placesHTTP/1.1
        Host:api.example.com
        Content-Type:application/json
        
        {
            "version" : "1.0" 7
        }

    如果客户端的请求是JSON格式的，实现起来倒是不难。如果是Content-Type是image/png     或者是text/csv呢，这就不好处理。

    或者把参数放到Query参数里：
        POST/places?version=1.0HTTP/1.1
    这种情况下，如果是POST请求呢，一些框架在POST请求的时候会直接忽略GET参数，虽然这有悖于HTTP协议，但是在POST请求带入GET参数，还是让人非常困惑的。

3. 自定义请求头
        GET/placesHTTP/1.1
        Host:api.example.com
        BadApiVersion:1.0
   
   看起来好像挺好的，但是这绝对代码中的坏味道。为了让缓存系统能够正确低返回果，我们Response必须是这个样子的：

        HTTP/1.1200OK
        BadAPIVersion:1.1
        Vary:BadAPIVersion
    
    如果不指定Vary，像[varnish][5]缓存系统是不知道如何缓存这样的请求的。另外，抛开这点不说，要知道这个HTTP还必须通过查阅文档才能了解，这也挺恼人的。关于Vary头，请参考[6]。

4. Content Negotiation
    这种方式是符合HATEOAS，也是相对最优雅的方式。[Github API][7]的Versioning就是通过这种方式实现的。

         Accept:application/vnd.github.user.v4+json
         Accept:application/vnd.github.user+json;version=4.0

    这种方式，对HATEOAS和缓存都非常友好。唯一可能有点麻烦的就是实现, 主流的框架都没有处理自动根据请求的Content-Type处理的机制。

代码实现
=======

1. 对于URI的实现策略，最直接有效的实现方式就是通过多个代码库实现API的多个版本。部署时候把不同的版本部署到不同的server上。像这样：
    - 1.0/master
    - 1.0/develop
    - 2.0/master
    - 2.0/develop

    有一个非常好的[Git Flow][8]可以参考。

2. 对于Content Negotiation需要同一个代码库的实现，这个是重点。

   ***以Rails为例***。 Rails本身有一个很好的Gem叫做[versionist][9]实现了这个功能。
    为了更好的理解其实现，我们可以自己实现以下。

首先是路由，给v1版本设置namespace
------------------------------

/config/routes.rb
```
Store::Application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :products
    end
  end
  
  resources :products
  root to: 'products#index'
end
```

接着是v1的实现
------------
/app/controllers/api/v1/products_controller.rb
```
module Api
  module V1
    class ProductsController < ApplicationController
      respond_to :json
      
      def index
        respond_with Product.all
      end
      
      def show
        respond_with Product.find(params[:id])
      end
      
      def create
        respond_with Product.create(params[:product])
      end
      
      def update
        respond_with Product.update(params[:id], params[:products])
      end
      
      def destroy
        respond_with Product.destroy(params[:id])
      end
    end
  end
end
```

然后，v2版本需要变更这个API
------------------------
###数据库变更###
/config/db/migrations/201205230000_change_products_released_on.rb
```
class ChangeProductsReleasedOn < ActiveRecord::Migration
  def up
    rename_column :products, :released_on, :released_at
    change_column :products, :released_at, :datetime
  end

  def down
    change_column :products, :released_at, :date
    rename_column :products, :released_at, :released_on
  end
end
```
这样V2版本已经能正确返回的修改后的结果了。
###保留V1###
```
module Api
  module V1
    class ProductsController < ApplicationController
      class Product < ::Product
        def as_json(options={})
          super.merge(released_on: released_at.to_date)
        end
      end
      
     respond_to :json
        # Actions omitted
    end
  end
end
```

这里两个版本的ProductsController有很多重复代码，不太符合DRY原则。旧版本的代码的保留通常了为了兼容，总有那么一天旧版本的API废止了，那么重构就不值得了。如果确实觉得需要，可以把共通的行为提取到超类里面。

***[完整示例][10]***

[1]: http://blog.steveklabnik.com/posts/2011-07-03-nobody-understands-rest-or-http#i_want_my_api_to_be_versioned
[2]: http://www.informit.com/articles/article.aspx?p=1566460
[3]: http://stackoverflow.com/questions/389169/best-practices-for-api-versioning
[4]: http://www.lexicalscope.com/blog/2012/03/12/how-are-rest-apis-versioned/
[5]: https://github.com/varnish/Varnish-Cache
[6]: https://www.imququ.com/post/vary-header-in-http.html
[7]: https://developer.github.com/v3/media/
[8]: http://nvie.com/posts/a-successful-git-branching-model/
[9]: https://github.com/bploetz/versionist
[10]: http://railscasts.com/episodes/350-rest-api-versioning?view=asciicast
