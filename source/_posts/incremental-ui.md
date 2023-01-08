---
title: Another approach to declarative incremental UI in Rust
date: 2022/11/16
---

Recently I have had some new(maybe not at all) but simple(possibly flawed) ideas about how we architect the UI framework in an incremental and declarative way in Rust.

The reason why declarative is important is we developers do not want to write imperative and manual view updating or create code to sync the state change with our view. So we all agreed in the industry nowadays we could use  View = F(State) to directly express their relationship in a functional way.

The reason why incremental is essential is performance matters. View = F(State). is nice, but 99% time in runtime, our view receives events from the platform and modifies the application state in a very incremental behavior. The state change is super small. Rerun the view create function is wasteful and impossible(because we may lose the local state stored on view).

View = F(State) is the hardest part to be performant. So obviously, working with View-delta = F(State-delta) seems a better choice, we just apply the delta-view to the view and apply the delta-state to the state.

The question here is how could we get the delta of the state.

- depend on a powerful language runtime like js. watch the state change directly. (getter setter patch, proxy, GC).
    - It’s hard to do in rust, not ergonomic I suppose.
- diffing the state every time when updating.
    - Most rust frameworks do this but I do not favor it.
        - To get reasonable performance when the state scales, the immutable data structure is a must, which means:
            - Intrusive modification to the application state and how you mutate them
        - Unavoidable performance overhead. (diff cost, cache locality? allocation pressure?)
- compiler magic recognizes the state writes and triggers a reaction.
    - I don’t think it’s possible in practice
- ..

I have to ask myself. I mutated the states, why can not I know what I have mutated? Why do I have to pay the runtime cost to know it? 

My answer to this question is simple. **We could just direct record the state change in an explicit way.** This sounds like cheating, but reasonable. The cost we have to pay is we can not directly use the assigning operators or just call mutate function to mutate the states but create another data structure to express what mutation I will apply.

---

In rust let's see how we abstract this concept:

```rust
pub trait IncrementAble {
  /// `Delta` should be strictly the smallest atomic modification unit of `Self`
  /// atomic means no invalid states between the modification
  type Delta;
  type Error: Debug; // apply delta may be failed due to invalid inputs

  /// apply the mutations to the data
  fn apply(&mut self, delta: Self::Delta) -> Result<(), Self::Error>;

  /// generate a sequence of the delta. use this we could 
  /// fully copy the Self
  ///
	/// This method is crucial
  fn expand(&self, cb: impl FnMut(Self::Delta));
}

/// type alias for convenience
pub type DeltaOf<T> = <T as IncrementAble>::Delta;
```

To implement this for primitive types we could just:

```rust
impl IncrementAble for u32 {
  type Delta = Self;

  type Error = ();

  fn apply(&mut self, delta: Self::Delta) -> Result<(), Self::Error> {
    *self = delta;
    Ok(())
  }

  fn expand(&self, mut cb: impl FnMut(Self::Delta)) {
    cb(self.clone())
  }
}
```

To implement this for container types we could:

```rust
pub enum VecDelta<T: IncrementAble> {
  Push(T),
  Remove(usize),
  Insert(usize, T),
  Mutate(usize, DeltaOf<T>), // note this: how we composing inner delta type
  Pop,
}

impl<T: IncrementAble + Default> IncrementAble for Vec<T> {
  type Delta = VecDelta<T>;
  type Error = (); // omit

  fn apply(&mut self, delta: Self::Delta) -> Result<(), Self::Error> {
    match delta {
      VecDelta::Push(value) => {
        self.push(value);
      }
      VecDelta::Remove(index) => {
        self.remove(index);
      }
      VecDelta::Insert(index, item) => {
        self.insert(index, item);
      }
      VecDelta::Pop => {
        self.pop().unwrap();
      }
      VecDelta::Mutate(index, delta) => {
        let inner = self.get_mut(index).unwrap();
        inner.apply(delta).unwrap();
      }
    };
    Ok(()) 
  }

// note here we call every vec item recursively
// to fully expand the vec (actually virtually rebuild from empty default)
  fn expand(&self, mut cb: impl FnMut(Self::Delta)) {
    for (i, v) in self.iter().enumerate() {
      cb(VecDelta::Push(T::default()));
      v.expand(|d| { 
        cb(VecDelta::Mutate(i, d));
      })
    }
  }
}
```

Compose type(the type we define in our apps usually)

```rust
struct TodoItem {
  name: String,
  finished: bool,
}

/// actually convert the product type to sum type.
enum TodoItemChange {
  Finished(bool),
  Name(String),
}

impl IncrementAble for TodoItem {
  type Delta = TodoItemChange;
  type Error = ();
  fn apply(&mut self, delta: Self::Delta) -> Result<(), Self::Error> {
    match delta {
      TodoItemChange::Finished(v) => self.finished.apply(v)?,
      TodoItemChange::Name(v) => self.name.apply(v)?,
    }
    Ok(())
  }

  fn expand(&self, mut cb: impl FnMut(Self::Delta)) {
    cb(TodoItemChange::Name(self.name.clone()));
    cb(TodoItemChange::Finished(self.finished.clone()));
  }
}
```

This could be directly derived by macros:

```rust
#[derive(Incremental)]
struct TodoList {
  list: Vec<TodoItem>,
}

#[derive(Incremental, Default)]
struct TodoItem {
  name: String,
  finished: bool,
}
```

Your app state is the composition of these types. By trait work, we could simply implement incrementable to any(almost) T automatically. 

The T::Delta is a strict complex sum type. Let’s reason about it from a low-level perspective.

- Memory cost
    - almost as same as the delta except for the enum tags which map the hierarchy of your app states hierarchy. I suppose it’s not a big problem. For tree-like states, you could just use the flattened tree container to avoid the recursive behavior in the type level.
- Delta constructing cost
    - user should usually only construct sub-delta because the user only works with sub-state. For example, the user modifies a sub-state by creating a sub-delta, then the sub-delta pop up to the parent wrapping type and is wrapped in the parent delta type. Pop and pop until it converts into the root app state delta type.
    - Stack data move trivial cost exists but should be optimized by the compiler as long as the call inlines. I suppose.
- Apply cost
    - The complex root state delta is used to consume in actual mutation apply or checked by the other code(like when we do view state update). This boil down to how pattern matching works in rust, which is zero abstraction cost.
- App state expand all delta cost
    - this is as same as the delta constructing cost. But this point is our expand method is passing a callback to implementation. So deltas are created one by one in a deep call stack and void any unnecessary heap allocations(if our delta type does not contain heap allocation)
    - Like the code above, the Vec’s implementation of Incrementable. We have to create an item default before we apply item mutations. This cost is ok and essential I think. The default value should be trivial for containers and small types

**My conclusion is: Working with(store, passing, using incrementable API) delta types is good(in ergonomics and performance way), even if the delta type and state type are super complex(has a deep hierarchy in type level).**

---

 Now let’s see how we use this delta abstraction in incremental UI: 

(note, in this part, we omit super large details such as view implementations(such as rendering layout, platform event), and only focus on how state changes incremental update the view)

This trait is to express our view

```rust
// represent the mouse keyboard, file, network events
pub struct PlatformEvent; 

/// View type could generics over any state T, as long as the T could provide
/// given logic for view type.
trait View<T>
where
  T: IncrementAble,
{
  /// sepcial event type
  type Event;

  /// In event loop handling, view type received platform events such as mouse move keyboard events,
  /// and decide should react to it or not, if so, generate the mutation for state or emit
  /// the self::Event for further outer side handling. see ViewReaction.
  ///
  /// In the View hierarchy, event's mutation to state will pop up to the root, wrap the mutation to
  /// parent state's delta type. and in update logic, consumed from the root
	///
	/// the reaction is handled in callback
  fn event(&mut self, model: &T, event: &PlatformEvent, cb: impl FnMut(ViewReaction<Self::Event, T>));

  /// update is responsible for mapping the state delta to view property change
  /// the model here is unmodified by delta.
  fn update(&mut self, model: &T, delta: &T::Delta);
}

pub enum ViewReaction<V, T: IncrementAble> {
  /// emit self-special event
  ViewEvent(V),
  /// do state mutation
  StateDelta(T::Delta),
}
```

the event method is the user update logic in the below graph.  Mapping the platform events to state(T) delta(T::Delta) or just emit custom ViewEvent for outer parent view(and the parent view will handle the child view event by converting to parent state change as well).

![Screen Shot 2022-11-14 at 20.46.27.png](/images/incremental-ui/27.png)

The delta(purple below) will be poped to the app root level state and then using the view’s update method to propagate changes. The view instance, I mean any other view instance(not restricted to the one which produces the delta), could check the super complex delta type and decide if the change is useful for their own. So the property binding simply becomes pattern matching the delta and extracts the real delta.

![Screen Shot 2022-11-14 at 20.47.06.png](/images/incremental-ui/06.png)

how state delta bind to view property: the same idea as above.

![Screen Shot 2022-11-14 at 20.47.01.png](/images/incremental-ui/01.png)

---

I will next demonstrate some pseudo-code for this concept.

If there is a simple Todo app. The final UI code could be like this: The style is similar to the druid, but rely on the incrementable trait to work with the state instead of the druid’s data trait. You can see how we express explicit delta and bind delta to properties here.

```rust
// define the state.
#[derive(Incremental)]
struct TodoList {
  list: Vec<TodoItem>,
}

#[derive(Incremental, Default)]
struct TodoItem {
  name: String,
  finished: bool,
}

fn todo_list_view() -> impl View<TodoList, Event = ()> {
  Container::wrap(
    TextBox::placeholder("what needs to be done?")
      .on(submit(|value| { // handle view events
        TodoListChange::List(VecDelta::Push(TodoItem { 
          name: value, // you can see we explicitly express delta
          finished: false,
        }))
      })),
    List::for_by(todo_item_view)
      .lens(lens!(TodoList::list)) // yes we use druid lens to extract sub state
      .on(inner(|event| TodoListChange::List(VecDelta::Remove(event.index)))), // handle view events
  )
}

enum TodoItemEvent {
  DeleteSelf,
}

fn todo_item_view() -> impl View<TodoItem, Event = TodoItemEvent> {
  Container::wrap(
    Title::name(bind!(Name)), // bind the name delta to name property
    Toggle::status(bind!(Finished)), // ditto
    Button::name("delete")
      .on(click(|event, item| TodoItemEvent::Delete)),
  )
}
```

In the view’s building blocks, we can see how they implement the view trait:

```rust
struct TextBox<T: IncrementAble> {
  texting: String,
// this is the binding: check the delta if the delta we care and extract
  text_binding: Box<dyn Fn(&DeltaOf<T>) -> Option<&String>>,
}

enum TextBoxEvent {
  Submit(String),
}

impl<T: IncrementAble> View<T> for TextBox<T> {
  type Event = TextBoxEvent;

  fn event(&mut self, model: &T, event: &PlatformEvent, mut cb: impl FnMut(ViewReaction<Self::Event, T>)) {
    let react = false;
    // omit processing logic
    if react {
      cb(ViewReaction::ViewEvent(TextBoxEvent::Submit(self.texting.clone())))
    } 
      
  }
  fn update(&mut self, model: &T, delta: &T::Delta) {
    if let Some(new) = (self.text_binding)(&delta) {
      self.texting = new.clone(); // do the real view state sync
    }
  }
}
```

The property binding: check the delta if it’s what I want:

```rust
impl<T: IncrementAble> TextBox<T> {
  pub fn with_text(mut self, binder: impl Fn(&DeltaOf<T>) -> Option<&String> + 'static) -> Self {
    self.text_binding = Box::new(binder);
    self
  }
}

fn _test(text: TextBox<TodoItem>) {
  text.with_text(bind!(DeltaOf::<TodoItem>::Name)); // may be could shorter?
}

#[macro_export]
macro_rules! bind {
  ($Variant: path) => {
    |delta| {
      if let $Variant(name) = delta {
        Some(&name)
      } else {
        None
      }
    }
  };
}
```

Another important one is the container type or the combinator type, which composites a complex view type to form our UI. Here we use the List as an example.

```rust

// V is Itemlist's View item type
struct List<V> {
  views: Vec<V>,
  build_item_view: Box<dyn Fn() -> V>,
}

impl<V> List<V> {
  pub fn for_by(view_builder: impl Fn() -> V + 'static) -> Self {
    Self {
      views: Default::default(),
      build_item_view: Box::new(view_builder),
    }
  }
}

// wrapper for the inner view event
struct EventWithIndex<T> {
  event: T,
  index: usize,
}

impl<T: IncrementAble + Default, V: View<T>> View<Vec<T>> for List<V> {
  type Event = EventWithIndex<V::Event>;

  fn event(
    &mut self,
    model: &Vec<T>,
    event: &PlatformEvent,
    mut cb: impl FnMut(ViewReaction<Self::Event, Vec<T>>),
  ) {
    for (i, view) in self.views.iter_mut().enumerate() {
      view.event(model.get(i).unwrap(), event, |e| {
// this is so called delta pop
        cb(match e {
          ViewReaction::ViewEvent(e) => ViewReaction::ViewEvent(EventWithIndex { index: i, event: e }),
          ViewReaction::StateDelta(delta) => ViewReaction::StateDelta(VecDelta::Mutate(i, delta)),
        })
      });
    }
  }

  fn update(&mut self, model: &Vec<T>, delta: &DeltaOf<Vec<T>>) {
    match delta {
      VecDelta::Push(v) => {
        self.views.push((self.build_item_view)());
        let pushed = self.views.last_mut().unwrap();
        v.expand(|d| pushed.update(v, &d));
      }
      VecDelta::Remove(_) => todo!(),// omit
      VecDelta::Insert(_, _) => todo!(),
      VecDelta::Mutate(index, d) => {
        let v = model.get(*index).unwrap();
        let view = self.views.get_mut(*index).unwrap();
        view.update(v, d)
      }
      VecDelta::Pop => {
        self.views.pop();
      }
    }
  }
}
```

The trick part in List is the vector’s push in the update method. Imagine our todo item view type watched the delta of todo item fields, but we never watch the entire todo item create. This is also the tricky part when you hand-write view update logic: you have to both handle the creating and updating, which is not declarative. To solve this problem we use Incremtable’s expand method. **Expand method provides a way to normalize all state creating to state updating. So we could only work with the update. The declarative core is not over the create but the update.**

The List example also shows the potential ways to support item move or swap. In other frameworks, item states should be labeled uniquely to maintain a stable identity across the list and to recognize moving action when updating. Now, if we simply support the swap or move mutation variant in delta type. this problem does even not exist.

Another question is, the state in view trait is immutable, but who should modify the state in the end? Of course the state owner. The state owner could be the root view to hold our app state. The important is, any state delta will be pop the state owner and then processed, which guaranteed any state delta dependent no matter how far they are, they could listen to the right same change. The immutable state access in the trait also guaranteed the state will not be changed accidentally, only the owner has the right to do the delta broadcast and real modification.

**Fundamentally, the complex-composed delta type effectively encodes the data mutation path at the type level on delta type. As long as the mutation path is unique, the mutation could be safely watched.**

This restriction means the interior mutable types should be ruled out for our state. Because an `Rc<RefCell<T>>`  could be shared in your state tree, which means you could modify the state by a different visit path from the root state type. It’s impractical for the view, the delta watcher side to handle all modification paths. So keep your state a clean tree. if you need to use graph data structure, simply use the arena solution but not build graph node by hand with `rc refcell`. I think this restriction also applies to immutable data structures.

I don’t know if there are mathematic tools to describe the T versus T::Delta, the state space versus state change space. Working with the T maybe should have some homotopy way to work with T::Delta? If we purely think in T::Delta space, seems a promising way to solve the reactive incremental UI problem.

---

In summary, the data flow is entirely based on the delta, so the cost of view update strictly follows the delta quantities and how complicated we process the delta. The view trait guided strong view type composition is suitable for compiler optimization to produce efficient UI code.

Compare to other solutions, especially the diffing. Besides the better performance potential, we don’t need to change our data type intrusively and depend on the opaque immutable containers with sophisticated algorithms, just define a new mutation interface for old data, a new way to modify our state. Compare to the reactive watch solutions, this approach provides a full-detail, hierarchy app-level source of truth delta, which could be watched and react at any granularity.

The Incrementable trait could also provide encapsulation for other advance incremental compute subsystems. Bridge them all together and create complex applications with incremental performance implications.

---

Further interesting direction: The incrementable trait could be modified to support the reversed delta. The reversed delta is the opposite of the delta, effectively canceling out the original delta. This could abstract the incremental undo-redo ability.

```rust
/// Not all types can impl this kind of reversible delta
pub trait ReverseIncrementAble: IncrementAble {
  /// return reversed delta
  fn apply_rev(&mut self, delta: Self::Delta) -> Result<Self::Delta, Self::Error>;
}
```

Not all types could provide an effective (not clone all(except for immutable data structure)) implementation, because information may be lost in the transformation.

This ability enables us to record all the app state history in a compact efficient way. We could use the delta to do something interesting like time traveling and debugging.