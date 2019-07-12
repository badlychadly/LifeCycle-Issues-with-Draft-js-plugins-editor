# LifeCycle-Issues-with-Draft-js-plugins-editor

As I have been working on a React Rails blogging web app using Draft-js and Draft-js-plugins I ran into an issue when I wanted to use the same component that is redering the editor for the `posts/new` route and the `posts/:id/edit` routes. For the `posts/new` route a blank editor is rendered, but for the edit route I need to pass the correct post object and use the body of that object to pass into `convertFromRaw` to display the existing data inside the editor. My initial attempt was to declare the state the same way for the editor with existing content as the empty editor and then use the Component Lifecycle method `componentDidMount` and check the available props to for an existing post.

## The Issue
The problem with making changes to the editor in componentDidMount is that the decorators in the plugins for the plugin Editor will not get added. Since our component contains the plugin editor component once the `render()` method is called the plugin editor starts its own lifecycle. The plugin editor uses the now legacy lifecycle methods of `componentWillMount` and `componentWillRecieveProps`. In order for the plugins decorators to be added to the editorState `componentsWillRecieveProps` depends on the state that was passed to the `onChange()` in `componentWillMount`, but inbetween these two methods our components `componentDidMount()` is called also calling onChange and overriding the editorState that contained the plugins decorators data. Take a look at the following code from the plugin editor.
```
{
    key: 'componentWillMount',
    value: function componentWillMount() {
      var decorator = (0, _resolveDecorators2.default)(this.props, this.getEditorState, this.onChange);

      var editorState = _draftJs.EditorState.set(this.props.editorState, {
        decorator: decorator
      });

      this.onChange((0, _moveSelectionToEnd2.default)(editorState));
    }
  }, {
    key: 'componentWillReceiveProps',
    value: function componentWillReceiveProps(next) {
      var curr = this.props;
      var currDec = curr.editorState.getDecorator();
      var nextDec = next.editorState.getDecorator(); // If there is not current decorator, there's nothing to carry over to the next editor state

      if (!currDec) return; // If the current decorator is the same as the new one, don't call onChange to avoid infinite loops

      if (currDec === nextDec) return; // If the old and the new decorator are the same, but no the same object, also don't call onChange to avoid infinite loops

      if (currDec && nextDec && getDecoratorLength(currDec) === getDecoratorLength(nextDec)) return;

      var editorState = _draftJs.EditorState.set(next.editorState, {
        decorator: currDec
      });

      this.onChange((0, _moveSelectionToEnd2.default)(editorState));
    }
  }
```
This looks intimidating but just pay attention to the names of the functions and that they both call onChange. Now in our component we may have something like this:
```
 onChange = (editorState) => {
        this.setState((prevState, props) => {
            return {editorState}
        });
    }

componentDidMount() {
  this.onChange(EditorState.moveFocusToEnd(this.state.editorState))
}
```
Consider that in the `onChange()` method we call `setState()` which is asynchronous, and onChange adds the changes from our component and the plugin editor Component. Let's walk through these existing methods and their execution stack one by one.
1. componentWillMount is called passing the editorState with the added decorators to onChange.
2. Our Components componentDidMount is ran passing a new editorState without the decorators to the onChange method. 
3. onChange from componentWillMount is ran setting the state with the decorators.
4. onChange from our componentDidMount is ran overriding the the state that was just set with the decorators.
5. componentWillRecieveProps in the plugins editor is ran and the editorState it depends on doesn't contain the decorators it depends on, so it fails the first conditional statement and calls return;.

## Solution
The workaround I found for this is to avoid having lifeCycle methods in my component to alter the editorState and instead use a conditional ternary operator to set the state.
```
state = !!this.props.post ? ({
        editorState: EditorState.createWithContent(convertFromRaw(JSON.parse(this.props.post.body)))
    }) : ({
        editorState: EditorState.createWithContent(convertFromRaw(rawBlock))
    })
```
If the post exists in props then the EditorState will be created with that posts content. In order for this to work though you need to pass the needed post to the component being redered by the route, otherwise it will not be available at the time we are trying to declare the state.
