---
sidebar_label: Optimizing React components
title: Optimizing rendering React components
hide_title: true
---

# Optimizing rendering React components

MobX is very fast, [often even faster than Redux](https://twitter.com/mweststrate/status/718444275239882753).

But here are some tips to get most out of React and MobX. Most tips apply to React in general and are not specific for MobX.

Note that while it's good to be aware of these patterns, usually your application
will be fast enough if you don't worry about them at all. Prioritize performance only
when it's an actual issue!

## Use many small components

`observer` components will track all values they use and re-render if any of them changes.
So the smaller your components are, the smaller the change they have to re-render; it means that more parts of your user interface have the possibility to render independently of each other.

## Render lists in dedicated components

This is especially true when rendering big collections.
React is notoriously bad at rendering large collections as the reconciler has to evaluate the components produced by a collection on each collection change.
It is therefore recommended to have components that just map over a collection and render it, and render nothing else:

Bad:

```javascript
const MyComponent = observer(({ todos, user }) => (
    <div>
        {user.name}
        <ul>
            {todos.map(todo => (
                <TodoView todo={todo} key={todo.id} />
            ))}
        </ul>
    </div>
))
```

In the above listing React will unnecessarily need to reconcile all `TodoView` components when the `user.name` changes. They won't re-render, but the reconcile process is expensive in itself.

Good:

```javascript
const MyComponent = observer(({ todos, user }) => (
    <div>
        {user.name}
        <TodosView todos={todos} />
    </div>
))

const TodosView = observer(({ todos }) => (
    <ul>
        {todos.map(todo => (
            <TodoView todo={todo} key={todo.id} />
        ))}
    </ul>
))
```

## Don't use array indexes as keys

Don't use array indexes or any value that might change in the future as key. Generate ids for your objects if needed.
See also this [blog](https://medium.com/@robinpokorny/index-as-a-key-is-an-anti-pattern-e0349aece318).

## Dereference values late

When using `mobx-react` it is recommended to dereference values as late as possible.
This is because MobX will re-render components that dereference observable values automatically.
If this happens deeper in your component tree, less components have to re-render.

Fast:

```javascript
<DisplayName person={person} />
```

Slower:

```javascript
<DisplayName name={person.name} />
```

There is nothing wrong with the latter, but a change in the `name` property will, in the first case, trigger only the `DisplayName` to re-render, while in the latter, the owner of the component has to re-render. If rendering the owning component is fast enough (usually it will be!), that approach will work well.

### Function props

[🚀] You may notice that to deference values late you have to create lots of small observer components where each is customized to render some different part of data, for example:

```javascript
const PersonNameDisplayer = observer(({ person }) => <DisplayName name={person.name} />)

const CarNameDisplayer = observer(({ car }) => <DisplayName name={car.model} />)

const ManufacturerNameDisplayer = observer({ car}) => (
    <DisplayName name={car.manufacturer.name} />
))
```

This quickly becomes tedious if you have lots of data of different shape. An alternative is to use a function that returns the data that you want your `*Displayer` to render:

```javascript
const GenericNameDisplayer = observer(({ getName }) => <DisplayName name={getName()} />)
```

Then, you can use the component like this:

```javascript
const MyComponent = observer(({ person, car }) => (
    <>
        <GenericNameDisplayer getName={() => person.name} />
        <GenericNameDisplayer getName={() => car.model} />
        <GenericNameDisplayer getName={() => car.manufacturer.name} />
    </>
))
```

This approach will allow `GenericNameDisplayer` to be reused throughout your application to render any name, and you still keep component re-rendering
to a minimum.
