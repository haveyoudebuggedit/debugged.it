---
slug: react-hexagonal-architecture
guid: https://pasztor.at/blog/react-hexagonal-architecture/
authors:
- janos
categories:
- Technology
date: "2018-08-26T00:00:00Z"
publishDate: "2018-08-26T00:00:00Z"
summary: ReactJS is a workhorse of frontend development, but it doesn't do so well in terms of S.O.L.I.D. Can we change it by adopting a classic approach?
images:
- posts/hexagonal-architecture-react.png
preview: posts/hexagonal-architecture-react.jpg
tags:
- Software Development
title: Hexagonal React
---

When looking at ReactJS as a modern JavaScript solution for a UI, a lot of people seem to have a problem organizing
their code. Let's take a deep dive and see if we can make it better. 

{{% tip %}}
**In a hurry?** [Get the code right now](https://github.com/janoszen/react-hexagonal-architecture)
{{% /tip %}}

## What is React?

Writing frontend code in JavaScript has been a painful experience ever since I started doing web development some 
20 years ago. Mostly due to browser incompatibilities, but also because there was no workable component framework for
it. Sure enough, ExtJS / now Sencha was available, but it was more painful to use than not.

Then React came along, attempting to solve the problem. React used a next generation version of JS, which you would
*transpile* to work on currently available browsers. In React you could create reusable components such as this: 

```
class MyBlogComponent {
    render() { 
        return <div className="blogentry">
          <div className="blogentry__title">{this.props.title}</div>
          ...
        </div>;
    }
}
```

And then you could use these components in other components:

```
class FeaturedBlogPost {
    render() {
        return <MyBlogComponent title="Featured article" />;
    }
}
```

Each of these components has two parts: props and state. Props are things that are passed externally. In our example 
above "title" is a prop. State on the other hand is a components internal state. A good example for internal state 
would be when you need to load data from a server in a component and then display it.

These components ultimately create what's called a virtual DOM. It's a tree, just like the real DOM, but it is much,
much faster. So fast, in fact, that it can recompute the entire DOM for every state change.

React then takes the virtual DOM and compares the different versions in order to figure out which parts of the real
DOM need updating.

## The FLUX/REDUX architecture

As you might imagine the initial implementation wasn't exactly flawless. Each component needed to maintain their own
internal state, and when two different components used the same data, there would often be inconsistencies as the data
would be loaded from the server at different times.

Trying to work around this problem the FLUX or REDUX architecture was born. The two are slightly different, but we'll
treat them as one for the purposes of this article.

The core idea is that the application has a central *state* that stores everything in the application. When a 
component triggers an action, that action would then change the state (through action creator functions), which in turn
would change the rendered content of the components using that piece of the global state.

{% xneato %}
digraph flux {
  component [
    shape=box
    label="React component"
    pos = "0,3!"
  ]
  action [
    shape=box
    label="Action"
    pos = "1,2!"
  ]
  reducer [
    shape=box
    label="Reducer"
    pos = "1,1!"
  ]
  store [
    shape=box
    label="Store"
    pos = "0,0!"
  ]
  
  component->action
  action->reducer
  reducer->store
  store->component
}
{% endxneato %}

This, of course, provides a centralized architecture to split out state handling from the individual components, making
them presentational in nature only.

However, as you may have guessed, this architecture also has its fair share of problems. For one, it has a *centralized*
state that is basically a giant god object storing *all* the data in the application. My second problem is centered
around the un-S.O.L.I.D-ness of the solution. In order to change one piece of the application you'd have to touch (and
test) various code parts in the application, and a change in the data structure could have a wide range of consequences.

## The hexagonal architecture

One architecture that worked [extremely well for me](/blog/the-cookie-cutter-architecture) is the hexagonal, or as I
like to call it, the cookie cutter architecture. This architecture makes use of *services* to carry out actions and
*entites* to pass around data in a structured format.

For a long time I saw no way to recreate this architecture in Javascript since it did not have dependency injection, or
static typing for that matter. Recently, however, I've been playing around with Typescript and Blueprint as a component
framework for React and I managed to find a way to build a similar system.

First of all we are going to start with our business logic. For example, we are going to build an
`AuthenticationService`. In contrast to my previous articles where I described backend architectures, the primary place
of storing *state* is going to be the service layer. As such the authentication service has a couple of local fields:

```
class AuthenticationService {
    private accessToken: IAccessToken|null = null;
    private account: IAccount|null = null;
}
```

These entities can be defined as simple interfaces like this:

```
interface IAccessToken {
    readonly id: string;
    readonly accountId: string;
    readonly expires: LocalDateTime;
}
```

When transpiled to JavaScript these will be represented by regular objects. Since we are building a service we will also
need some dependencies which can be injected through the constructor as expected:

```
class AuthenticationService {
    //...
    public constructor(
        readonly authenticationBackend: IAuthenticationBackendApi
    ) {
    }
}
```

And finally the meat of the matter, let's add the authentication method:

```
class AuthenticationService {
    //...
    public authenticate = (username: string, password: string): Promise<boolean> => {
       
    };
}
```

As you can see we are working with promises because the authentication process takes time. When the promise completes,
the authentication process has either completed or failed. We can then implement a login form like this:

```
class AuthenticationDialog
    extends React.Component<IAuthenticationDialogProps, IAuthenticationDialogState>
{
    render = () : JSX.Element => {
        return <form>
          <input ... />
          <input ... />
          <button onClick={this.onLogin} />
        </form>
    };

    private onLogin = () => {
        this.setState({
            loading: true
        });
        const self = this;
        this.props.authenticationService
            .authenticate(this.state.username,this.state.password)
            .catch(() => {
                self.setState({
                    loading: false
                });
                //Authentication process failure
            })
            .then((result: boolean) => {
                self.setState({
                    loading: false
                });
                if (result) {
                    //Successful login
                } else {
                    //Failed login
                }
            })
    };
}
```

### Handling events

Until now we only reacted to events. How do you then update a React component based on events that are happening?
Let's stick with the authentication example. Let's look at our authentication service:

```
class AuthenticationService {
    //...
    registerAuthenticationChangeListener(listener: IAuthenticationChangeListener):void {
    }
    deregisterAuthenticationChangeListener(listener: IAuthenticationChangeListener):void {
    }
}
```

So we add these two new functions that allow a component to register itself as an event listener. In a component you can
then do the following:

```
class RequireAuthenticated
    extends React.Component
    implements IAuthenticationChangeListener
{
    public state : IAuthenticationContextState = {
        isAuthenticated: false
    };

    public componentDidMount = (): void => {
        this.props.authenticationService.registerAuthenticationChangeListener(this);
        this.onAuthenticationChange();
    };

    public onAuthenticationChange = ():void => {
        this.setState({
            isAuthenticated: this.props.authenticationService.isAuthenticated()
        });
    };

    public render = () => {
        return this.state.isAuthenticated?this.props.children:[];
    }
}
```

As you can see this component only shows its contents when the user is authenticated. As the component comes up, it 
registers itself with the authentication service to be notified when the state changes. This will allow the DOM tree
to change when a state change happens.

### Constructing the component tree

You may notice that the authentication dialog gets the authentication service as a prop. So when you want to create
this authentication dialog, you'll have to do it like this:

```
const authService = new AuthenticationService();
const authDialog = <AuthenticationDialog authenticationService={authService} />
```

This, obviously, makes the process of creating the component tree a bit messy. To work around that we'll use factories:

```
class AuthenticationDialogFactory {
    public constructor(
        readonly authenticationServiceFactory: AuthenticationServiceFactory
    ) {

    }

    public create = () : JSX.Element => {
        return <AuthenticationDialog
            authenticationService={this.authenticationServiceFactory.create()}
        />
    }
}
```

This factory will then be used in our `index.jsx` file to construct the root app.

## Get the code

Now that you have a good understanding what's going on, you can
[grab the code from GitHub](https://github.com/janoszen/react-hexagonal-architecture).

Additionally you could extend this solution by adding React.DI.

## Conclusion

During my trials I found that it is ridiculously easy to write and debug code like this. It's more bytes, but it saves
you a ton of headache when debugging stuff. However, it's not the only viable solution.