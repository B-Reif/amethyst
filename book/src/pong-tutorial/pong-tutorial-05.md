# Winning Rounds and Keeping Score

Our last chapter ended on a bit of a cliffhanger. What happens when our ball
reaches the left or right edge of the screen? It just keeps going! 😦

In this chapter, we'll fix that by putting the ball back into play after it
leaves either side of the arena. We'll also add a scoreboard and keep track of
who's winning and losing.

## Winning and Losing Rounds

So let's fix the big current issue; having a game that only works for one
round isn't very fun. We'll add a new system that will check if the ball has
reached either edge of the arena and reset its position and velocity. We'll also
make a note of who got the point for the round.

First, we'll add a new module to `systems/mod.rs`

```rust
pub mod winner;
```

Then, we'll create `systems/winner.rs`:

```rust
# mod pong {
#   pub struct Ball {
#       pub radius: f32,
#       pub velocity: [f32; 2],
#   }
# 
#   pub const ARENA_WIDTH: f32 = 100.0;
#   pub const ARENA_HEIGHT: f32 = 100.0;
# }
# 
use amethyst::{
    core::{
        ecs::{ParallelRunnable, System},
        transform::Transform,
    },
    ecs::{IntoQuery, SystemBuilder},
};

use crate::pong::{Ball, ARENA_HEIGHT, ARENA_WIDTH};

pub struct WinnerSystem;

// NEW
impl System for WinnerSystem {
    fn build(self) -> Box<dyn ParallelRunnable> {
        Box::new(
            SystemBuilder::new("WinnerSystem")
                .with_query(<(&mut Ball, &mut Transform)>::query())
                .write_component::<Ball>()
                .write_component::<Transform>()
                .build(
                    move |_commands,
                          world,
                          (),
                          query| {
                        let (mut ball_world, mut _remaining) = world.split_for_query(query);

                        for (ball, transform) in query.iter_mut(&mut ball_world) {
                            let ball_x = transform.translation().x;
                            let did_hit = if ball_x <= ball.radius {
                                // Right player scored on the left side.
                                println!("Player 2 Scores!");
                                true
                            } else if ball_x >= ARENA_WIDTH - ball.radius {
                                // Left player scored on the right side.
                                println!("Player 1 Scores!");
                                true
                            } else {
                                false
                            };
                            if did_hit {
                                // Reset the ball.
                                ball.velocity[0] = -ball.velocity[0];
                                transform.set_translation_x(ARENA_WIDTH / 2.0);
                                transform.set_translation_y(ARENA_HEIGHT / 2.0);
                            }
                        }
                    },
                ),
        )
    }
}
```

Here, we're creating a new system, joining on all `Entities` that have a `Ball`
and a `Transform` component, and then checking each ball to see if it has
reached either the left or right boundary of the arena. If so, we reverse
its direction and put it back in the middle of the screen.

Now, we need to add our new system to `main.rs`, and we should be able to
keep playing after someone scores and log who got the point.

```rust
# use amethyst::{
    assets::LoaderBundle,
#   core::transform::TransformBundle, ecs::World, input::StringBindings, prelude::*,
#   window::DisplayConfig,
#   input::InputBundle,
# };
# 
# mod systems {
#   use amethyst::core::ecs::{System, World};
# 
#   pub struct PaddleSystem;
#   impl System for PaddleSystem {
#       fn build(mut self) -> Box<dyn ParallelRunnable> {}
#   }
#   pub struct MoveBallsSystem;
#   impl System for MoveBallsSystem {
#       fn build(mut self) -> Box<dyn ParallelRunnable> {}
#   }
#   pub struct BounceSystem;
#   impl System for BounceSystem {
#       fn build(mut self) -> Box<dyn ParallelRunnable> {}
#   }
#   pub struct WinnerSystem;
#   impl System for WinnerSystem {
#       fn build(mut self) -> Box<dyn ParallelRunnable> {}
#   }
# }
# 
# fn main() -> amethyst::Result<()> {
#   let path = "./config/display.ron";
#   let config = DisplayConfig::load(&path)?;
#   let input_bundle = amethyst::input::InputBundle::new();
# 
    let mut dispatcher = DispatcherBuilder::default();
#   dispatcher
#       .add_bundle(LoaderBundle)
#       // Add the transform bundle which handles tracking entity positions
#       .add_bundle(TransformBundle)
#       .add_bundle(
#           InputBundle::new().with_bindings_from_file(app_root.join("config/bindings.ron"))?,
#       )
#       // We have now added our own systems, defined in the systems module
#       .add_system(PaddleSystem)
#       .add_system(BallSystem)
#       .add_system(BounceSystem)
        // -- snip --
        .add_system(WinnerSystem);
#   let assets_dir = "/";
#   struct Pong;
#   impl SimpleState for Pong {}
#   let mut game = Application::new(assets_dir, Pong, game_data)?;
#   Ok(())
# }
```

## Adding a Scoreboard

We have a pretty functional Pong game now! At this point, the least fun thing
about the game is that players have to keep track of the score themselves.
Our game should be able to do that for us.

In this section, we'll set up UI rendering for our game and create a scoreboard
to display our players' scores.

First, let's add the UI rendering in `main.rs`. Add the following imports:

```rust
use amethyst::ui::{RenderUi, UiBundle};
```

Then, add a `RenderUi` plugin to your `RenderBundle` like so:

```rust
# use amethyst::{
#   ecs::World,
#   prelude::*,
#   renderer::{types::DefaultBackend, RenderingBundle},
#   ui::RenderUi,
# };
# fn main() -> Result<(), amethyst::Error> {
#   let mut dispatcher = DispatcherBuilder::default();
#   dispatcher
        .add_bundle(
            RenderingBundle::<DefaultBackend>::new()
                // ...
                .with_plugin(RenderUi::default()),
        )?;
#   Ok(())
# }
```

Finally, add the `UiBundle` after the `InputBundle`:

```rust
# use amethyst::ui::UiBundle;
# use amethyst::{ecs::World, input::InputBundle, prelude::*};
# fn main() -> Result<(), amethyst::Error> {
#   let display_config_path = "";
#   struct Pong;
#   let mut dispatcher = DispatcherBuilder::default();
#   dispatcher
        .add_bundle(UiBundle::<u32>::default())?
#;
# 
#   Ok(())
# }
```

We're adding a `RenderUi` to our `RenderBundle`, and we're also adding the
`UiBundle` to our game data. This allows us to start
rendering UI visuals to our game in addition to the existing background and
sprites.

> u32?
> **Note:** We're using a `UiBundle` with type `StringBindings` here because the
> `UiBundle` needs to know what types our `InputHandler` is using to map `actions`
> and `axes`. So know that your `UiBundle` type should match your
> `InputHandler` type. You can read more about those here: [UiBundle][ui-bundle],
> [InputHandler][input-handler].

Now we have everything set up so we can start rendering a scoreboard in our
game. We'll start by creating some structures in `pong.rs`:

```rust
/// ScoreBoard contains the actual score data
#[derive(Default)]
pub struct ScoreBoard {
    pub score_left: i32,
    pub score_right: i32,
}

/// ScoreText contains the ui text components that display the score
pub struct ScoreText {
    pub p1_score: Entity,
    pub p2_score: Entity,
}
```

> Don't glimpse over the `#[derive(Default)]` annotation for the `ScoreBoard` struct!

`ScoreBoard` is a container that will allow us to keep track of each
player's score. We'll use this in another module later in this chapter, so we've
gone ahead and marked it as public (same with `ScoreText`). `ScoreText` is also
a container, but this one holds handles to the UI `Entity`s that will be
rendered to the screen. We'll create those next:

```rust
use amethyst::ui::{Anchor, LineMode, TtfFormat, UiText, UiTransform};

# pub struct Pong;
# 
#  pub struct ScoreText {
#      pub p1_score: Entity,
#      pub p2_score: Entity,
#  }
impl SimpleState for Pong {
    fn on_start(&mut self, data: StateData<'_, GameData>) {
        let StateData {
            world, resources, ..
        } = data;
#       let world = data.world;
        // --snip--

        initialize_scoreboard(world, resources);
    }
}
// ...

/// initializes a ui scoreboard
fn initialize_scoreboard(world: &mut World, resources: &mut Resources) {
    resources.insert(ScoreBoard::default());

    let font = {
        let loader = resources.get::<DefaultLoader>().unwrap();
        loader.load("font/square.ttf")
    };
    let p1_transform = UiTransform::new(
        "P1".to_string(),
        Anchor::TopMiddle,
        Anchor::TopMiddle,
        -50.,
        -50.,
        1.,
        200.,
        50.,
    );
    let p2_transform = UiTransform::new(
        "P2".to_string(),
        Anchor::TopMiddle,
        Anchor::TopMiddle,
        50.,
        -50.,
        1.,
        200.,
        50.,
    );

    let p1_score = world.push((
        p1_transform,
        UiText::new(
            Some(font.clone()),
            "0".to_string(),
            [1., 1., 1., 1.],
            50.,
            LineMode::Single,
            Anchor::Middle,
        ),
    ));

    let p2_score = world.push((
        p2_transform,
        UiText::new(
            Some(font),
            "0".to_string(),
            [1., 1., 1., 1.],
            50.,
            LineMode::Single,
            Anchor::Middle,
        ),
    ));

    resources.insert(ScoreText { p1_score, p2_score });
}
```

Here, we add some UI imports and create a new `initialize_scoreboard` function,
which we'll call in the `on_start` method of the `Pong` game state.

Inside `initialize_scoreboard`, we first create a default ScoreBoard and insert it
into our world's resources. Next we load up a font which we've saved to `assets/font/square.ttf`([download][font-download]). We load the font as a resource in the
world, and then save the handle to a `font` variable (which we'll use to create
our `UiText` components).

Next, we create a transform for each of our two scores by giving them a unique
id (`P1` and `P2`), a UI `Anchor` at the top middle of our window, and then
adjust their global `x`, `y`, and `z` coordinates, `width`, `height`, and
`tab-order`.

After creating the `font` and `transform`s, we'll create an `Entity` in the
world for each of our players' scores, with their `transform` and a `UiText`
component (with a `font` handle, initial `text`, `color`, and `font_size`).

Finally, we initialize a `ScoreText` structure containing each of our UI
`Entity`s and add it as a resource to the world so we can access it from our
`System`s later.

If we've done everything right so far, we should see `0` `0` at the top of our
game window. You'll notice that the scores don't update yet when the ball makes
it to either side, so we'll add that next!

## Updating the Scoreboard

All that's left for us to do now is update the UI whenever a player scores a
point. You'll see just how easy this is with our `ECS` design. All we have to do
is modify our `WinnerSystem` to access the players' scores and update them
accordingly:

```rust
# mod pong {
#   pub struct Ball {
#       pub radius: f32,
#       pub velocity: [f32; 2],
#   }
# 
#   #[derive(Default)]
#   pub struct ScoreBoard {
#       pub score_left: i32,
#       pub score_right: i32,
#   }
# 
#   pub struct ScoreText {
#       pub p1_score: Entity,
#       pub p2_score: Entity,
#   }
# 
#   pub const ARENA_WIDTH: f32 = 100.0;
#   pub const ARENA_HEIGHT: f32 = 100.0;
# }
# 
use amethyst::{
#   core::{
#       ecs::{ParallelRunnable, System},
#       transform::Transform,
#   },
#   ecs::{IntoQuery, SystemBuilder},
    // --snip--
    ui::UiText,
};

use crate::pong::{Ball, ScoreBoard, ScoreText, ARENA_HEIGHT, ARENA_WIDTH};

pub struct WinnerSystem;

impl System for WinnerSystem {
    fn build(self) -> Box<dyn ParallelRunnable> {
        Box::new(
            SystemBuilder::new("WinnerSystem")
                .with_query(<(&mut Ball, &mut Transform)>::query())
                .with_query(<&mut UiText>::query())
                .write_component::<Ball>()
                .write_component::<Transform>()
                .write_component::<UiText>()
                .write_resource::<ScoreBoard>()
                .read_resource::<ScoreText>()
                .build(
                    move |_commands,
                          world,
                          (score_board, score_text),
                          (balls_query, edit_query)| {
                        let (mut ball_world, mut score_world) = world.split_for_query(balls_query);

                        for (ball, transform) in balls_query.iter_mut(&mut ball_world) {
                            let ball_x = transform.translation().x;
                            let did_hit = if ball_x <= ball.radius {
                                // Right player scored on the left side.
                                // We top the score at 999 to avoid text overlap.
                                score_board.score_right = (score_board.score_right + 1).min(999);
                                if let Ok(text) =
                                    edit_query.get_mut(&mut score_world, score_text.p2_score)
                                {
                                    text.text = score_board.score_right.to_string();
                                }
                                true
                            } else if ball_x >= ARENA_WIDTH - ball.radius {
                                // Left player scored on the right side.
                                // We top the score at 999 to avoid text overlap.
                                score_board.score_left = (score_board.score_left + 1).min(999);
                                if let Ok(text) =
                                    edit_query.get_mut(&mut score_world, score_text.p1_score)
                                {
                                    text.text = score_board.score_left.to_string();
                                }
                                true
                            } else {
                                false
                            };
                            if did_hit {
                                // Reset the ball.
                                ball.velocity[0] = -ball.velocity[0];
                                transform.set_translation_x(ARENA_WIDTH / 2.0);
                                transform.set_translation_y(ARENA_HEIGHT / 2.0);
                                // Print the score board.
                                println!(
                                    "Score: | {:^3} | {:^3} |",
                                    score_board.score_left, score_board.score_right
                                );
                            }
                        }
                    },
                ),
        )
    }
}
# fn main() {}
```

We've added a fair few changes here, so let's go through them. First, we want to
be able to read and write our scores, so we add the `UiText` storage, which
holds all `UiText` components, to our `SystemData`. We'll want to select our
players' scores from that, so we also add the `ScoreText` structure which holds
handles to the `UiText` components that we want. Finally, we add the
`ScoreBoard` resource so we can keep track of the actual score data.

We're using `write_resource` here to pull in the `ScoreBoard` instead of with
`with_query (&mut ...)` because we want mutable access to `ScoreBoard`, which is
not a collection of components but rather a single resource item.

We also use `read_resource` to access the `ScoreText` resource immutably. Again,
`ScoreText` is a single resource item rather than a collection of components.

Inside our `run` method (after updating the signature to match our `SystemData`
changes), we replace the `println!` statements with code that will update our
`UiText` components. We first update the score stored in `score_board` by
adding 1 to it and clamping it to not exceed `999` (mostly because we don't want
our scores to overlap each other in the window). Then, we use the `edit_query`
to get a mutable reference to our `UiText` component. Lastly, we set the text 
of the `UiText` component to the player's score, after converting it to a string.

## Summary

And that's it! Our game now keeps track of the score for us and displays it at
the top of our window.

![Pong Game with Scores](../images/pong_tutorial/pong_05.png)

Now don't go just yet, because, in the next chapter, we'll make our Pong game
even better by adding sound effects and even some music!

[font-download]: https://github.com/amethyst/amethyst/blob/main/examples/pong_tutorial_05/assets/font/square.ttf?raw=true
[input-handler]: https://docs.amethyst.rs/master/amethyst_input/struct.InputHandler.html
[ui-bundle]: https://docs.amethyst.rs/master/amethyst_ui/struct.UiBundle.html
