# This was another approach which I came accross

This was honestly seemed super interesting and passibly a bit more comprehensive and better but I didnt get a change to play with it.
It came from [this stack overflow thread](https://stackoverflow.com/questions/47579343/angularjs-directive-into-react-app)
by a guy named Sebastian. It got down voted but not sre why.
Here is the link he posted to the sample [code](https://react-angularjs-bridge.herokuapp.com/) 

I looked through his source files on the site and formatted what I found to be a little more readable and understandable.
Here is the code:

```js
import React from 'react';
import angular from 'angular';

// import React from 'react';
// import ReactAngularBridge from '../ReactAngularBridge/ReactAngularBridge';

/* ****************************************************************/

// interface IAppComponentState {
//   angularComponentEnabled: boolean;
//   logsStack: Array<{message: string, prefix: string, time: Date}>
// }

class App extends React.Component {
  state = {
    angularComponentEnabled: false,
    logsStack: []
  };


  toggleAngularComponent = () => {
    const angularComponentEnabled = !this.state.angularComponentEnabled;

    this.setState({angularComponentEnabled});

    this.log(`Toggled angular to state ${angularComponentEnabled}`)
  };

  log(message, prefix = 'react') {
    const logsStack = this.state.logsStack; // TSWTF
    const time = new Date();

    logsStack.push({message, prefix, time});

    this.setState({logsStack});

    return this;
  }

  render() {
    const bindings = {
      message: 'Toto, I\'ve a feeling we\'re not in Kansas anymore',
      onLog: (message) => this.log(message, 'angular'),
      onChange: () => this.log('Notified about a state change')
    };

    const appName = 'angularWidget';
    const appTemplate = '<angular-widget message="message" on-change="onChange()" on-log="onLog(message)"></angular-widget>';

    const logStack = this.state.logsStack.slice().reverse();

    return (
      <div className="app">
        <div className="content">
          <section className="react-section">
            <img src={logo} className="logo" alt="logo"/>
            <h3 className="title">react application</h3>
            <div>
              <button onClick={this.toggleAngularComponent}>Toggle angular component</button>
            </div>
          </section>

          <section className="angular-section">
            {this.state.angularComponentEnabled &&
            <ReactAngularBridge appName={appName} template={appTemplate} bindings={bindings}/>}
          </section>
        </div>

        <aside className="aside">
          <code className="logs-section">
            {logStack.map((log, index) => (
              <p key={index} className="log-entry">
                <span className="index">{log.time.toLocaleTimeString()}</span>
                <span className={`prefix ${log.prefix}`}>{log.prefix}</span>
                <span className="message">{log.message}</span>
              </p>
            ))}
          </code>
        </aside>
      </div>
    );
  }
}
/ ********************************************************************************/



// interface IComponentProps {
//   appName: string; // The angular module name
//   template: string; // The angular root component considering bindings are set on $rootScope `<my-component></my-component>`
//   bindings?: IComponentBindings; // The optional bindings object
// }

// interface IComponentBindings {
//     [key:string]: any
// }

class ReactAngularBridge extends React.Component {
  ref = React.createRef();

  $rootScope;

  $shadowRoot;

  get $element() {
    return this.ref.current ? this.ref.current : null;
  }

  componentDidMount() {
    // this.createApp().bootstrap();
  // createApp() { // : ReactAngularBridge
    if (!this.$element) throw new Error(`Could not create the angularJS application due to a missing element reference`)  //return this.onElementMissing();

    angular.module(this.props.appName)
        .run(['$rootScope', ($rootScope) => {
            this.$rootScope = $rootScope;

            //this.updateBindings();
            const bindings = this.props.bindings || {};

            // Reflect.ownKeys(bindings).forEach( key => this.setBinding(key, bindings[key]));
            // for each key on the bindings object, set the same key and a property on the $root scope 
            Reflect.ownKeys(bindings).forEach( key => this.$rootScope[key] = bindings[key] );
        }]);

        // setBinding(key, value) {
        //   this.$rootScope[key] = value;
      
        //   return this;
        // }

    this.$shadowRoot = this.$element.attachShadow({ mode: 'closed' }); // TODO createShadowRoot for IE compatibility

    this.$shadowRoot.innerHTML = this.props.template;

  //   return this;
  // }
  
  // bootstrap() {
    if (!this.$element) throw new Error(`Could not create the angularJS application due to a missing element reference`);;
    if (!this.$shadowRoot.firstElementChild) throw new Error(`Could not create the angularJS application due to a missing element reference`);;

    angular.bootstrap(this.$shadowRoot.firstElementChild, [this.props.appName]);

  //   return this;
  // }
  }

  componentDidUpdate(prevProps) { // : IComponentProps
    return this.updateBindings();
  }

  componentWillUnmount() {
      this.destroyNgApp();
  }

  // createApp() { // : ReactAngularBridge
  //   if (!this.$element) throw new Error(`Could not create the angularJS application due to a missing element reference`)  //return this.onElementMissing();

  //   angular.module(this.props.appName)
  //       .run(['$rootScope', ($rootScope) => {
  //           this.$rootScope = $rootScope;

  //           this.updateBindings();
  //       }]);

  //   this.$shadowRoot = this.$element.attachShadow({ mode: 'closed' }); // TODO createShadowRoot for IE compatibility

  //   this.$shadowRoot.innerHTML = this.props.template;

  //   return this;
  // }

  // bootstrap() {
  //   if (!this.$element) return this.onElementMissing();
  //   if (!this.$shadowRoot.firstElementChild) return this.onElementMissing();

  //   angular.bootstrap(this.$shadowRoot.firstElementChild, [this.props.appName]);

  //   return this;
  // }

  destroyNgApp() {
    this.$rootScope.$destroy();

    if (this.$shadowRoot) {
      while (this.$shadowRoot.firstChild) {
        this.$shadowRoot.removeChild(this.$shadowRoot.firstChild);
      }
    }

    return this;
  }

  updateBindings() {
    const bindings = this.props.bindings || {};

    Reflect.ownKeys(bindings).forEach( key => this.setBinding(key, bindings[key]));

    return this;
  }

  setBinding(key, value) {
    this.$rootScope[key] = value;

    return this;
  }

  // onElementMissing()  {
  //   throw new Error(`Could not create the angularJS application due to a missing element reference`);

  //   return this; // ☠ The caller has to implement Error Boundaries to prevent it
  // }

  render() {
    return (
      <div ref={this.ref}/>
    );
  }
}

export default ReactAngularBridge;
```