## Handling Templates in Angular with Webpack
We are using webpack to bundle all of our angular code up now and so there are some changes 
that are needed for specifying template locations. Instead of specifying a templateURL and using the template cache
we are going to be inlining our templates where they are needed with webpack. Consider the below component webpack config and 
component definition: 

```js
// webpack config
...
  module: {
    rules: [{
      /* Exports HTML as a string. This is used for inlining our angular templates
      for directives and components. Note that we must no longer use templateUrl
      within our angular code. Instead use template : require('./path/to/file').
      Webpack will use this loader to inline the template at that spot so we will
      no longer need templateCache.*/
      test: /\.html$/, use: ['html-loader']
      // production add: options: { minimize: true }
    }]
...
```

```js
angular.module('main')
  .component('sample',{
    controller: sampleController,
    template: require('./location/of/template.html')
  })
``` 
With the use of Webpack's html-loader and by doing a require('template/location'), when webpack bundles up the code, it will go out 
and grab that template at build time and inline it right there. All reference to templateURL have been removed and replaced with 
```js
template: require(...)
```
except for two which are in client.js which use ngtemplate-loader.

### ngTemplate-loader
There is one area which we were not able to inline our templates and that was for CSM CommonWeb's RowUpdateManager which is used in both the
client/PINManager as well as in the client/contactManager. The controller which initializes the RowUpdateManager is the adminController
and you can see the code there. Because the RowUpdateManager was not built to accept templates (only templateURL's) and it is in a separate 
library, I brought in a webpack loader which would allow us to use the templateURL's. Find it in package.json and node_modeules under 
[ngtemplate-loader](https://developer.aliyun.com/mirror/npm/package/ngtemplate-loader).
The best way to handle templates in AngularJS with webpack is to inline them like described above. However, there are 
still some usecases where templateURLs are needed, such as ours. That is likely the reason the ngtemplate-loader package isn't very well supported and 
doesnt have a ton of use overall, yet there are still recent updates and maintenence on the package. My reason for reaching for this package
was because I didnt want to alter CSMCommonWeb.
By putting html-loader in our webpack config as the default loader for .html modules, if we required the template which we wanted to use
teamplateURL for, Webpack would inline the template just like described above. To override that behavior and to choose which loader to use at
build time, we specify the loader within our require like so:
```js
  templateURL:  require(`ngtemplate-loader!./contactManager/contactManager.template.html`)
``` 
The two cases of using templateURL's are in the clientTemplateConstants defined in the client.js file. 
```js
export const clientTemplatesConstants = {
  CONTACT_MANAGER:       require(`ngtemplate-loader!./contactManager/contactManager.template.html`),
  DISTRICT_PIN_MANAGER:  require('ngtemplate-loader!./PINManager/districtPINManager.template.html')
  };
```