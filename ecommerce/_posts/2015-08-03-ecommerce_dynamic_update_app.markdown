---
layout: post
title:  "动态APP更新"
date:   2015-08-03 19:09:26
categories: ecommerce
---

#App 动态更新
* 资料
 * https://github.com/markmarijnissen/cordova-app-loader
 * https://github.com/murlex/UpdatableApp


* 注意事项
 * 因为项目使用ionic，内部集成angular，一般

```
<body ng-app="yourApp">
 ...
</body>
```


 但由于使用cordova-app-loader后，我们一般在bootstrap.js中动态加载所需的js
 
```
 <script src="lib/cordova-app-loader/dist/cordova-app-loader-complete.js"></script>
 <script type="text/javascript"
            manifest="manifest.json"
            src="lib/cordova-app-loader/dist/bootstrap.js"></script>
```
 
 此时所需的自定义的js,如 angular的[app.js/controller.js/service.js]尚未载入，如果直接使用ng-app会异常。 
 
 可以在自己的app.js中手动初始化解决

```javascript
angular.element(document).ready(function () {
    angular.bootstrap(document, ['starter']);
});
```

 * bootstrap.js 主要是为了动态加载js与css，不处理html，而我们希望动态更改app内的网页内容。这里我们通过gulp-angular-templatecache将各个angular template打包到一个统一的js。
 
 使用gulp打包:

```
var templateCache = require('gulp-angular-templatecache');

gulp.task('default', function () {
    gulp.src('app/**/*.html')
        .pipe(templateCache({ module:'templatesCache', standalone:true} ))
        .pipe(gulp.dest('www/templates/'));
    gulp.src(['app/js/controllers.js', 'app/js/services.js', 'app/js/app.js'])
        .pipe(concat('app.js'))
        .pipe(gulp.dest('www/js/'));
});
```

* 血泪教训
 * 我自己环境搭地比较早，所以cordova是3.x版本的，项目也是在这个基础上开发，测试，发布到应用市场。
 * 在把整个工程环境共享给同事的过程中，发现同事本地无法运行，查了原因后，发现同事因为是最新安装，因此cordova版本为5.x，必须引入cordova-plugin-whitelist ，并做相应的配置(http-equiv="Content-Security-Policy" )才可成功运行。
 * 以为这样就大功告成了，后续的改动也直接发布到应用市场了。 但后续发现ios上动态更新正常，但android无法动态更新，花了2天的时间，最后发现仍然是cordova-plugin-whitelist禁止了cdvfile所致.
 *  在config.xml中修改

```
<allow-navigation href="*://*/*"/>
```

* 后续发现修改allow-navigation无法通过连接拨打电话，或发送短信。追查了一番后，修改为

```
<allow-navigation href="*"/>
  <allow-navigation href="cdvfile://*"/>
```

