---
layout: post
title: SolidJS Object Destructuring
date: 2024-06-03 11:09 -0400
---

As I noted in my previous post, I have in the past explored SolidJS. It's been a while, and since [Solid Start](https://start.solidjs.com/) hit 1.0, I decided to give it another try. While I am using the SolidStart for a project, I'm not doing any routing or using any complicated state. I'm keeping it simple by building a tic tac toe clone. However I ran into something pretty quickly that is admittedly [well documented](https://www.solidjs.com/guides/reactivity#considerations) but was not obvious to me coming from React.

Firstly, and this I was aware of, component functions are only executed once. So if you have something like:

```javascript
function MyChildComponent() {
  return <p>{new Date().getTime()}</p>;
}
```

It doesnt matter if the parent re-renders, this component will only display the time of the initial render. However, where this got weird for me was with props:

```javascript
function MyChildComponent({ time }) {
  return <p>{time}</p>;
}
```

SolidJS experts will likely see my issue here immediately, I did not as destructuring props is pretty common in React. From reading some comments on a [reddit post](https://www.reddit.com/r/solidjs/comments/18dyxsb/why_not_pass_accessor_function_to_child_components/) what I realized is that any destructuring whether in the function definition or the body with the props, SolidJS loses the proxying it has to the data source of those props. So it wont be able to receive updates. So one always has to define & use props like this:

```javascript
function MyChildComponent(props) {
  return <p>{props.time}</p>;
}
```

This is also true if you use the store functionality, as noted in this thread: [https://www.reddit.com/r/solidjs/comments/10awle8/development_experience_with_solidjs/](https://www.reddit.com/r/solidjs/comments/10awle8/development_experience_with_solidjs/)

Overall I'd say this is a fairly minor downside, but i do find it a little disappointing, as what's been causing me grief more and more with React is the complexity. React hooks & its abstraction makes it hard to know what's going on at times. Controlling updated around useEffect is rather complicated. SolidJS seems to go for a more obvious & simpler API, but it does have some gotchas like this. I'm sure such examples can be guarded with things like linting, but why can't we have nice & easy things?

Maybe the real answer is that front end state and re-rendering is a complicated problem, and there isn't a way to fully simplify it. Still I want to keep playing with this more, and see how it stands up to the other giants.
