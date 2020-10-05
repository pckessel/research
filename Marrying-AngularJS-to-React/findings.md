# Mounting AngularJS components inside of a React Applicaiton

## The Basics
In order to mount an Angular component or directive from a React component, we first need to have reference to the DOM
node in which we will mount the Angular app onto. We do this with a ref in React and more specifically, a [callbackRef](https://reactjs.org/docs/refs-and-the-dom.html).
On the same DOM node we set the innterHTML to be the component or directive we want to display.
Finally once the component mounts, we manually tell angular to bootstrap the module onto our node which we now have local reference to.
consider this example from UpdateLogin:

```js
/* We're grabbing angular off the window as its already loaded onto the page with 
other angular packages in the ERWMaterPage.master. If we import angular here, it 
would work, but we'll get a few complaints in the console for trying to load angular
more than once, not to mention having to add it as a dependency to package.json */

// angular component definition
const ngUpdateLogin = window.angular.module("ngUpdateLogin", ['CSMCommonWeb'])
  .component('updateLogin', {
    controller: function(ChangeLoginDialog){
      ChangeLoginDialog.show()
    }
  })
  .name;

// react component definition
export default class UpdateLogin extends Component {
  constructor(){
    super();

    /* 
    Upon initial render, we'll snag that ref and store reference to it 
    in this.container. We will then bootstrap the angular module defined in 
    ngUpdateLogin.js file into the referenced DOM node.
    */
    this.container; 
  }

  componentDidMount(){
    window.angular.bootstrap(this.container, [ngUpdateLogin])
  }

  render() {
    return (
        <div 
          ref={container => this.container = container}
          dangerouslySetInnerHTML={{ __html: `<update-login></update-login>` }}
        />
    )
  };
};
```
In this example, you can see the controllers only responsibility is to show a dialog.
An interesting thing to point out though, is to get Angular to fire up the controller, I essentially had to
build an angular component that simply has a controller definition for the express purpose of being able
to pass in the html directive as the innerHTML of a DOMNode from React. 

As you can see from this example, we are simply using a React component as a container to grab a dom node which we 
then hand off to Angular to do whatever it wants with it. That node is now completely out of React's control and doesnt
have access to any of its lifecycle hooks. 
This is a very simple approach for stand alone components and works fairly well. Client Admin is a different beast.

---

## Client Admin Page
Client Admin is more complex than the above simple example as it uses the browser router to dynamically show different pages.
Consider this clientAdmin config:

```js
angular.module('main',['all-its-dependencies'])
  .config(function ($routeProvider) {
    var resolve = {
      auth: function (Authenticator) {
        return Authenticator.auth()
      },
      const: function (ERWConstants) {
        return ERWConstants.init()
      },
    }
    var logout = function (Authenticator) {
      Authenticator.logout()
    }

    $routeProvider
      .when('/ClientAdmin', {
        controller: 'AdminController',
        template: require('../ngClientAdmin/clientAdmin.template.html'),
        resolve: resolve,
      })
      .when('/ClientAdmin/client/:id', {
        controller: 'AdminClientController',
        template: require('../ngClientAdmin/client/client.template.html'),
        resolve: resolve,
      })
      .when('/ClientAdmin/person/:id', {
        controller: 'AdminPersonController',
        template: require('../ngClientAdmin/person/person.template.html'),
        resolve: resolve,
      })
      .when('/Logout', { controller: logout, template: '' })
      .when('/:newState', {
        controller: 'swfController',
        template: '',
        inSWF: true,
        resolve: resolve,
      })
      .otherwise({ redirectTo: '/Main' })
  })
```
Since we are using React to handle our app mavigation and we are using a React container to handle 
bootstrapping our Angular modules, we need to make some changes. 
This config block is going away so we wont have reference to which template uses which controller anymore so
The first change I made was to break up each route into its own angular component. See the below code for a
contrived example.

```js
window.angular.module('main', ['list-of-all-its-dependencies'])
  .component('admin', adminComponent)
  .component('adminPerson', adminPersonComponent)
  .component("adminClient", adminClientComponent)

const adminComponent = {
  template: require('./clientAdmin.template.html'),
  controller: adminController
};

const adminClientComponent = {
  template: require('./client.template.html'),
  controller: adminClientController
};

const adminPersonComponent = {
  template: require('./person.template.html'),
  controller: adminPersonController
};
```

With angular components now defined for each route, we can build out a React container with some functionality to 
handle routing the angular app appropriately.

React Container:

```js
export default class ClientAdminWrapper extends Component {
  constructor(){
    super();

    this.state = { component: `<admin></admin>`, nodeKey: 50 };

    this.changeRenderedComponent = this.changeRenderedComponent.bind(this);
    this.nodeReference; // placeholder until component renders
  }

  componentDidMount(){
    /********** Handle Routing between angular and React *********/
    // add listener to window to handle routing communication between Angular and React.
    // We are using hash router and hashchange is a browser api which fires on hashchange.
    window.addEventListener('hashchange', this.changeRenderedComponent)

    // get component and setState;
    this.changeRenderedComponent(); 

    /***** Initialize the Angular app ******/
    window.angular.bootstrap(this.nodeReference, [ Main ])
  }

  componentDidUpdate(){
    // this.changeRenderedComponent listens for Window location changes and then updates state 
    // with a new nodeKey and component to render.
    // Bootstrap that new component onto our newly created domNode. (key change) 
    window.angular.bootstrap(this.nodeReference, [ Main ])
  }

  componentWillUnmount(){
    window.removeEventListener('hashchange', this.changeRenderedComponent);
  }

  changeRenderedComponent() {
    // new key value to force React to recreate a DOM node & force rerender.
    const nodeKey = Math.floor((Math.random() * 100) + 1) 

    const hash = window.location.hash;

    let component = `<admin></admin>`; // default 
    if(hash.includes('client')) component = `<admin-client></admin-client>`;
    else if(hash.includes('person')) component = `<admin-person></admin-person>`;

    this.setState({ component, nodeKey })
  }

  render(){
    return (
      <div 
        key={this.state.nodeKey} 
        ref={node => this.nodeReference = node} 
        dangerouslySetInnerHTML={{ __html: this.state.component }}
      />
    )
  }
}
```

The state of our react component holds two things: component, which is an html directive string we want to set as our
local nodeReference's innerHTML, and nodeKey, which is just a random number between 1-100;
The component is pretty self explanitory; which component should be rendered at that time?
There is a little more to the nodeKey, though. To understand it better, lets first look that the local method called 
`changeRenderedComponent`. This function generates a new number betweeen 1 and 100, looks at the window location hash
and checks to see if it includes either 'client' or 'person'. and sets a local component variable to a directive string.
It then calls a React.Component method of setState which sets the state of the react component and causes react to re-run the render
method which is defined. 
When React rereders, it will only recreate pieces of the dom which it sees that have changed, and since we are
dangerously setting the innerHTML of a referenced dom node, React will not see that the innerHTML has changes since we are
explicitly opting out of React's functionality by setting it ourselves. Hence the name `dangerouslySetInnerHTML`.
So, upon rerender, React wont know what innerHTML of its div is and will not recognize that it has changed. By not recognizing 
that anything has changed, React will actually keep the same exact DOMNode in the tree. After the render method is run react calls
componentDidUpdate because the State had changed and then we try and Bootstrap the module again. But thats a problwm with angular. Angular
wont bootstrap the same module onto the same DOMNode. Here is the error: `[[ng:btstrpd] App Already Bootstrapped with this Element '&lt;div id="client-admin" class="ng-scope"&gt;' http://errors.angularjs.org/1.5.0/ng/btstrpd?p0=%26lt%3Bdiv%20id%3D%22client-admin%22%20class%3D%22ng-scope%22%26gt%3B]`

Thats a problem because we need to bootstrap angular with a new component everytime a route changes so we can get the controllers 
to run. The way we handle that is with the `key` prop that we give to the node. By changing the `key` prop to a random number everytime
we change the component, React will see that the DOM node is in fact different and it will tear it down and rebuild a new one. Now with
a new DOMNode, we can bootstrap angular again on a new node and it will run its controller as expected. 

Finally, when we navigate away from the page and React unmounts this component, we remove the even listnener from the window.

The main takeaways here are that Angular and React arent make to talk to each other so we need to use the thing which they both have in
common in order to glue them together; DOM Nodes and Browser APIs. We are using location changes to trigger React state updates which we
are then using to tell angular to run. Knowledge of React's diffing algorith lets us force it to rerender nodes when we need it to. 

The following articles and posts helped me to begin to understand the appraoch for mounting Angular to React:
https://softeng.oicr.on.ca/chang_wang/2017/04/17/Using-AngularJS-components-directives-in-React/
https://stackoverflow.com/questions/47579343/angularjs-directive-into-react-app
---

## Handling Templates in Angular with Webpack
We are using webpack to bundle all of our angular code up now (except Login) and so there are some changes 
that are needed for specifying template locations. Instead of specifying a templateURL and using the template cache
we are going to be inlining our templates where they are needed with webpack. Consider the below component definition: 
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
---

## Initializing ERWConstants
We are having to initialize the ERWConstants here manually due to how we are connecting up Angular to React.
Previously there was this config block in the main module:
```js
angular.module('main')
  .config(function ($routeProvider) {
    var resolve = {
      ...
      const: function (ERWConstants) {
        return ERWConstants.init()
      },
    }

    $routeProvider
      ...
      .when('/ClientAdmin/person/:id', {
        controller: 'AdminPersonController',
        template: require('../ngClientAdmin/person/person.template.html'),
        resolve: resolve,
      })
      ...
  })
```
We are no longer using the angular router and so the resolve functions never get run before the route is ready to 
be hit, leaving the ERWConstants uninitialized. The other appraoch I considered was to make the init function an IIFE so that 
as soon as main was bootstraped and the modules were being created, the init would automatically run, the problem with that 
was that the init wouldn't resolve until *after* this controller was initialized and run through which was causing the ERWConstants'
references in the RELATIONSHIPS array to be undefined. So the less sophisticated yet easier to reason about and understand way 
to do it is the way we are doing it in both the client.js and person.js files - Manually init and then reassign the local variables
after we know the values are there. 

### ERWContactRoles
The ERWContactRoles Factory is used in two separate controllers clientAdminController in client.js as well as in contactManagerController
in contactManager/index.js.
The factory is defined as so: 
```js
export function ERWContactRolesFactory (ERWConstants) {
  return 0,
    [
    { value: ERWConstants.CCR_PrimaryContact,    display: 'Primary Contact'},
    { value: ERWConstants.CCR_SecondaryContact,  display: 'Secondary Contact'},
    { value: ERWConstants.CCR_AuthorizedSigner,  display: 'Authorized Signer'},
    { value: ERWConstants.CCR_TechnicalContact,  display: 'Technical Contact'},
    { value: ERWConstants.CCR_BillContact,       display: 'Bill Contact'},
    { value: ERWConstants.CCR_NSLPContact,       display: 'NSLP Contact'},
    { value: ERWConstants.CCR_NSLPContactII,     display: 'NSLP Contact II'},
    { value: ERWConstants.CCR_BearSignerContact, display: 'Bear Signer Contact'},
    { value: ERWConstants.CCR_Superintendent,    display: 'Superintendent'},
    { value: ERWConstants.CCR_EPCAdministrator,  display: 'EPC Administrator'}
    ];
}
```
Due to the problems of ERWConstants not being initialized before each route is ready to be run like descibed above, the runtime binding of
all the the `value`'s in the array above will be `undefined`. In order to get around this, I have removed the factory definition and instead 
defined both of these arrays to be used *in* their local controllers. This means we have code duplication and identiacal array definitions in 
two spots. Obviously not ideal and should be watched out for when looking through the code. There are comments above each array indicating code
duplication.

---