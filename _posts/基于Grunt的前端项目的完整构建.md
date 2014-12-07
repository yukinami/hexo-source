title: 基于Grunt的前端项目的完整构建
tags:
  - 前端开发
date: 2014-12-07 14:25:54
---

本文主要简述一个前端项目使用grunt来进行构建的过程。项目本身是基于`RequireJS`的，样式则使用了`Less`来书写。

项目结构
=======

先来看一下项目整体的目录结构。

![项目结构](/img/CE761032-FAA5-4FFE-9FC7-F87BA7DFF423.jpeg)

Gruntfile
=========

通常我们需要grunt分别在开发，已经部署时为我们进行构陷。

运行时构建
---------
运行时构建需要的步骤如下，

``` JavaScript
  grunt.registerTask('serve', 'Compile then start a connect web server', function (target) {
    if (target === 'dist') {
      return grunt.task.run(['build', 'connect:dist:keepalive']);
    }

    grunt.task.run([
      'clean:server',
      'bower:install',
      'less',
      'concurrent:server',
      'configureProxies',
      'autoprefixer',
      'connect:livereload',
      'watch'
    ]);
  });
 ```

1. 清理上次的构建
2. 安装bower的依赖，这里安装到app的lib目录，以便`RequrieJS`进行引用
3. 编译`styles`目录下的`main.less`文件

		    less: {
		      dist: {
		          files: {
		              '<%= yeoman.app %>/styles/main.css': ['<%= yeoman.app %>/styles/main.less']
		          },
		          options: {
		              sourceMap: true,
		              sourceMapFilename: '<%= yeoman.app %>/styles/main.css.map',
		              sourceMapBasepath: '<%= yeoman.app %>/',
		              sourceMapRootpath: '/'
		          }
		      }
		    },

4. 复制`styles`目录下的css文件到`.tmp`目录。这里的`concurrent:server`只是并行的执行了`copy:styles`任务。

			styles: {
		        expand: true,
		        cwd: '<%= yeoman.app %>/styles',
		        dest: '.tmp/styles/',
		        src: '{,*/}*.css'
		    }

5. `configureProxies`是`grunt-connect-proxy`的任务，   由于开发阶段，前端项目是运行在node的服务器中，JS脚本和后台服务器并不在一个服务器，所以涉及脚本跨域问题，所以在开发阶段，需要配置一个代理服务器。
6. `autoprefixer` 用来解析`.tmp`目录下的CSS文件并且添加浏览器前缀到CSS规则里，处理后输出回styles目录。
7. 配置[grunt-contrib-connect][1]，支持livereload。这里需要配置两个地方：
	+ `grunt-contrib-connect`的`livereload`选项
		> Type: Boolean or Number
		> Default: false
		> 
		> Set to true or a port number to inject a live reload script tag into your page using connect-livereload.
		>
		> This does not perform live reloading. It is intended to be used in tandem with grunt-contrib-watch or another task that will trigger a live reload server upon files changing.
	+ `grunt-contrib-connect`的`middleware`选项
		middleware支持`livereload`，可以通过[connect-livereload][2]或者[grunt-connect-proxy][3]

  因为之前已经使用了`grunt-connect-proxy`，这里还是使用`grunt-connect-proxy`

          livereload: {
	        options: {
	          open: true,
	          base: [
	            '.tmp',
	            '<%= yeoman.app %>'
	          ],
	          middleware: function (connect, options) {
	            if (!Array.isArray(options.base)) {
	              options.base = [options.base];
	            }
	            var middlewares = [require('grunt-connect-proxy/lib/utils').proxyRequest];

	            options.base.forEach(function (base) {
	                grunt.log.warn(base);
	                middlewares.push(connect.static(base));
	            })
	            return middlewares;
	          }
	        }
	      }
8. 配置watch，当文件有变化时进行reload

部署时构建
----------
相较于运行时的构建，部署构建最主要的差别在于需要对资源文件进行合并，压缩等优化工作。

```
  grunt.registerTask('build', [
    // Clean dist and .tmp directories
    'clean:dist',
    // Run JSHint on all js files
    'jshint',
    // Install bower components into {xd.app}/lib
    'bower:install',
    // Compile LESS files into CSS
    'less',
    // Copy CSS files into .tmp
    'copy:styles',
    // Run autoprefixer on CSS files under .tmp
    'autoprefixer',
    // Run useminPrepare to generate concat.generated and cssmin.generated targets
    'useminPrepare',

    // Concat CSS files in .tmp
    'concat',
    // Copy concat css into dist
    'cssmin',
    // minify and copy the minified images into dist
    // TODO: imagemin has platform dependent issues; hence commenting out them for now.
    // 'imagemin',
    // Copy other necessary files into dist
    'copy:dist',

    // Now operate on dist directory
    // Static file asset revisioning through content hashing
    'filerev',
    // Rewrite based on revved assets
    'usemin',
    'htmlmin'
  ]);
 ```
 1. `clean:dist`和运行时构建类似，清理上一次的build构建。
 2. `jshint` 对js文件进行语法检查
 3. `bower:install` 同运行时构建
 4. `less` 同运行时构建
 5. `copy:styles` 同运行时构建
 6. `autoprefixer` 同运行时构建
 7. `useminPrepare` 任务更新gruntfile来设置一个配置好的转换流程。默认的对JS转换流程是concat以及uglifly。这个task需要需要Blocks来生成对应的grunt task。Blokcs的配置请参考[Blocks](https://github.com/yeoman/grunt-usemin#blocks)
	    useminPrepare: {
	      html: '<%= yeoman.app %>/index.html',
	      options: {
	        dest: '<%= yeoman.dist %>',
	        flow: {
	          html: {
	            steps: {
	              js: ['concat', 'uglifyjs'],
	              css: ['cssmin']
	            },
	            post: {}
	          }
	        }
	      }
	    },
 8. `concat`是由`useminPrepare`生成的task，进行js文件的合并。
 9. `cssmin`也是由`useminPrepare`生成的task，进行css文件的压缩。
10. `copy:dist`任务拷贝所有的资源到的最终需要部署的web项目的public目录中。
	      dist: {
	        files: [{
	          expand: true,
	          dot: true,
	          cwd: '<%= yeoman.app %>',
	          dest: '<%= yeoman.dist %>',
	          src: [
	            '*.{ico,png,txt}',
	            '.htaccess',
	            '*.html',
	            'views/{,*/}*.html',
	            'images/{,*/}*.{webp}',
	            'fonts/*',
	            'lib/**/*'
	          ]
	        }, {
	          expand: true,
	          cwd: '.tmp/images',
	          dest: '<%= yeoman.dist %>/images',
	          src: ['generated/*']
	        }, {
	          expand: true,
	          cwd: 'bower_components/bootstrap/dist',
	          src: 'fonts/*',
	          dest: '<%= yeoman.dist %>'
	        }]
	      },
11. `filerev`根据文件的内容生成修订版本号，用于文件缓存。
12. `usemin`执行两个步骤，替换源文件的资源引用为最终修订的文件。
	>First it replaces all the blocks with a single "summary" line, pointing to a file creating by the transformation flow.
	>Then it looks for references to assets (i.e. images, scripts, ...), and tries to replace them with their revved version if it can find one on disk
13. `htmlmin`进行最终的对HTML进行压缩

[1]: https://github.com/gruntjs/grunt-contrib-connect "grunt-contrib-connect"
[2]: https://github.com/intesso/connect-livereload    "connect-livereload"
[3]: https://github.com/drewzboto/grunt-connect-proxy#adding-the-middleware "grunt-connect-proxy"

