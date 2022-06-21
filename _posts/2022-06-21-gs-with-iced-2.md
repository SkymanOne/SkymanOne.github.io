---
layout: post
title: "GUI in Rust with iced #2: Composable Layout"
tags: [rust, iced, iced-tutorial]
---

This is the next issue of my series of tutorials for [iced](iced.rs) library.
Today we will learn how to do conditional rendering.


## Recap

In case you missed, in [previous tutorial]({% post_url 2022-06-13-gs-with-iced-1 %}) we covered the basics of Elm architecture, difference between retained and immediate modes and created the counter app. I strongly recommend you to complete the previous tutorial before starting if you haven't had experience with iced before.

## Conditional Rendering: Theory

Remember the *model-view-update* cycle? Conditional rendering is a technique that allows to draw our UI based on the state of our application.
Take a blog system as a example. We would show `Edit post` button, only when the authenticated user has the relevant role (editor or admin). In terms of the state of an application, we can say that we would display the button only when `(is_admin ||  is_editor) == true`. 

## Practical 

Let's reuse the code of our counter app from [previous tutorial]({% post_url 2022-06-13-gs-with-iced-1 %}) and add another page which will be shown upon the click on the button "Next page".

In `src` directory create another file called `main_page.rs` and let's define an empty structure to represent our page:

```rust
pub struct MainPage;
```

Now, let's source the module in our `main.rs` file, so we can reuse our page struct there:
```diff
  use iced::Settings;
  use iced::pure::widget::{Button, Column, Container, Text};
  use iced::pure::Sandbox;

+ mod main_page;
```

If the code above looks a little bit confusing to you, I recommend visiting [Chapter 7 of the Rust Book](https://doc.rust-lang.org/book/ch07-05-separating-modules-into-different-files.html).

For now, our page will just display a label *Hello from Page 2* for simplicity. We'll extend the functionality later.

Let's implement our struct by adding public `new()` and `view()` methods:

```rust 
impl MainPage {
    pub fn new() -> Self {
        MainPage
    }

    pub fn view(&self) -> Element<CounterMessage> {
        Container::new(Text::new("Hello from Page 2"))
            .width(Length::Fill)
            .height(Length::Fill)
            .center_x()
            .center_y()
            .into()
    }
}
```

And add relevant imports at the top of a file

```rust
use iced::{pure::{
    widget::{Container, Text},
    Element,
}, Length};

use crate::CounterMessage;
```
Okay, we need to figure out what's happening here:

* `pub fn new() -> Self` simply instantiates our view
* `pub fn view(&self) -> Element<CounterMessage>` builds our views and return a single `Element` that would produce messages of type `CounterMessage` from the previous tutorial. 

I'll explain why we do this in a moment.

Let's go back to `main.rs` and modify it.

First, add another enumeration `Views` that would indicate the current page to be shown.

```rust
#[derive(Debug, Clone, Copy)]
pub enum Views {
    Counter,
    Main
}
```

Our application state now needs to store the current page to be shown, so let's add it as a field to the `Counter` struct
```diff
 struct Counter {
     count: i32,
+    current_view: Views
 }
```

Now, rust compiler would ask you to modify `new()` method to include the initialisation of the field. Let's do it:

```diff
 fn new() -> Self {
     Counter { 
         count: 0,
+        current_view: Views::Main
     }
 }
```

Let's add another message which would notify our application to update the state, let's call it `ChangePage`

```diff
 #[derive(Debug, Clone, Copy)]
 pub enum CounterMessage {
     Increment,
     Decrement,
+    ChangePage(Views)
 }
```
Notice that we passing `Views` enum as a param of our enum case in order to let application know what views to be shown.

Next, let's cover this message case in our `update()` method

```diff
 fn update(&mut self, message: Self::Message) {
     match message {
         CounterMessage::Increment => self.count += 1,
         CounterMessage::Decrement => self.count -= 1,
+        CounterMessage::ChangePage(view) => self.current_view = view
     }
 }
```

We simple update the state of our application by assigning the `current_view` with the `view` that is passed by the message.

We now need to somehow produce this message. The interesting part start in `view()` method.

Let's create another button that will navigate to the page we created by producing relevant message
```rust
let navigate = Button::new(Text::new("Go to the next page")).on_press(CounterMessage::ChangePage(Views::Main));
```

and add it to the layout and add some spacing: 

```rust
let col = Column::new().push(incr).push(label).push(decr).push(navigate);
```

```diff
 fn view(&self) -> iced::pure::Element<Self::Message> {
     let label = Text::new(format!("Count: {}", self.count));
     let incr = Button::new("Increment").on_press(CounterMessage::Increment);
     let decr = Button::new("Decrement").on_press(CounterMessage::Decrement);
+    let navigate = Button::new(Text::new("Go to the next page")).on_press(CounterMessage::ChangePage(Views::Main));
-    let col = Column::new().push(incr).push(label).push(decr);
+    let col = Column::new().push(incr).push(label).push(decr).push(navigate).spacing(5);
     Container::new(col).center_x().center_y().width(iced::Length::Fill).height(iced::Length::Fill).into()
 }
```
The button simply produces `CounterMessage::ChangePage` message with the specified values of `Views` enum.

However, this is not it. Now, out application needs to know what view to show in the window.

In iced, every layout component implements `Widget` trait which can be converted into `Element<Message>` object. Essentially, the whole navigation in iced apps come down to drawing different groups of elements which are grouped by Layout components such as `Column`, `Row` and `Container`. 

Let's store the returned column in the variable:
```rust
let counter_layout = Container::new(col).center_x().center_y().width(iced::Length::Fill).height(iced::Length::Fill);
```

Note, that we remove `.into()` method. This is because we don't need to turn our widget into element yet. There is no inferred type yet.

We now need to instantiate our `MainPage`
```rust
let main_page_layout  = MainPage::new().view();
```
I took a shortcut there and also returned a layout of our page inlined, because that's what we interested in. If you are using rust-analyzer, you can see that it shows the type of our variable as `Element<CounterMessage, Renderer<...>>`

Now let's simply return the relevant vies based on value of `current_view`:

``` rust 
match self.current_view {
    Views::Counter => counter_layout.into(),
    Views::Main => main_page_layout
}
```

Ooops, rust borrow checker is now complaining with: `cannot return value referencing temporary value
returns a value referencing data owned by the current function`

This is because we try to instantiate our page struct and then return an element (from `.view()`) from its reference which will go out of scope as soon as we go through another *model-view-update* cycle. Hence, we need to own the data, we could replace `pub fn view(&self)` with `pub fn view(self)` in `main_page.rs`, but this would imply that `CounterMessage` must have `'static` lifetime which introduces unnecessary lifetime management.

So, instead let out application struct own the data by storing the page in state of our application:

in `main.rs`

```diff
struct Counter {
    count: i32,
    current_view: Views,
+   main_page: MainPage
}
```

```diff
 fn new() -> Self {
     Counter { 
         count: 0,
         current_view: Views::Main,
+        main_page: MainPage::new()
     }
 }
```

and now we can slightly modify our `view()` method in `main.rs` by removing 
```diff 
- let main_page_layout  = MainPage::new().view();
```

and using the stored `MainPage` instance

```diff
 match self.current_view {
     Views::Counter => counter_layout.into(),
-    Views::Main => main_page_layout.view() 
+    Views::Main => self.main_page.view()
 }
```

Let's see the final result from `cargo run`

![](/images/iced/one-way-nav.gif)


here is the code from `main.rs`

```rust
use iced::Settings;
use iced::pure::widget::{Button, Column, Container, Text};
use iced::pure::Sandbox;
use main_page::MainPage;

mod main_page;


fn main() -> Result<(), iced::Error> {
    Counter::run(Settings::default())
}

struct Counter {
    count: i32,
    current_view: Views,
    main_page: MainPage
}

#[derive(Debug, Clone, Copy)]
pub enum Views {
    Counter,
    Main
}

#[derive(Debug, Clone, Copy)]
pub enum CounterMessage {
    Increment,
    Decrement,
    ChangePage(Views)
}

impl Sandbox for Counter {
    type Message = CounterMessage;

    fn new() -> Self {
        Counter { 
            count: 0,
            current_view: Views::Counter,
            main_page: MainPage::new()
        }
    }

    fn title(&self) -> String {
        String::from("Counter app")
    }

    fn update(&mut self, message: Self::Message) {
        match message {
            CounterMessage::Increment => self.count += 1,
            CounterMessage::Decrement => self.count -= 1,
            CounterMessage::ChangePage(view) => self.current_view = view
        }
    }

    fn view(&self) -> iced::pure::Element<Self::Message> {
        let label = Text::new(format!("Count: {}", self.count));
        let incr = Button::new("Increment").on_press(CounterMessage::Increment);
        let decr = Button::new("Decrement").on_press(CounterMessage::Decrement);
        let navigate = Button::new(Text::new("Go to the next page")).on_press(CounterMessage::ChangePage(Views::Main));
        let col = Column::new().push(incr).push(label).push(decr).push(navigate).spacing(5);
        let counter_layout = Container::new(col).center_x().center_y().width(iced::Length::Fill).height(iced::Length::Fill);
        
        match self.current_view {
            Views::Counter => counter_layout.into(),
            Views::Main => self.main_page.view()
        }
    }
}
```


and `main_page.rs`

```rust
use iced::{pure::{
    widget::{Container, Text},
    Element,
}, Length};

use crate::CounterMessage;

pub struct MainPage;

impl MainPage {
    pub fn new() -> Self {
        MainPage
    }

    pub fn view(&self) -> Element<CounterMessage> {
        Container::new(Text::new("Hello from Page 2"))
            .width(Length::Fill)
            .height(Length::Fill)
            .center_x()
            .center_y()
            .into()
    }
}
```

## Explanation

If you haven't noticed, our `MainPage` does not implement any traits, it doesn't have methods that our `Counter` application has. It has absolutely arbitrary definition. We can even define it as enum and implement method on it.

This is the beauty of `iced` - **flexibility**. It doesn't impose any design patterns and architectural techniques on your application. 
You can define your pages and widgets in septate files and modules or keep them in the single file (like [Game of Life example](https://github.com/iced-rs/iced/blob/master/examples/pure/game_of_life/src/main.rs)).

Generally, iced application is structured around the fact, that you have a single point of truth. That is your application model that contains state, update and view implementation and centralised `Message` definition that your application produce.

In the next session we will look into how to make your pages produce messages of different type and integrate them into the global state of your application.

## Homework

I decided it would be useful to leave some open problems for you to work on:
* Currently, we only go to `MainPage` but not back, try to implement this
* There is a little bit of spaghetti-code in `main.rs`. Our counter page is mixed with general state of the application. Try to fix it.


If you have some feedback, comments or questions, I would love to hear them. Please reach out to me on discord: SkymanOne#9623