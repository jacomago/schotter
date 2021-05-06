# Schotter4: Animate the squares

For schotter4, we'll animate the squares, making them move and rotate over time. We want the changes to be smooth, so we need to move and rotate the squares just a bit each iteration.

There are several ways to do the animation; here is the strategy we'll use here: We'll keep the existing fields of the Stone struct (x, y, x_offset, y_offset, and rotation) to mean the current state of the stone. The view() function can then remain exactly the same. To animate the squares, we'll add some velocity fields: x_velocity, y_velocity, and rot_velocity, and add the values of these fields to x_offset, y_offset, and rotation in the update function to move and rotate the square. We'll also add a counter named cycles to track the number of iterations to use these velocity values before changing them.

So here's the augmented Stone struct:

```
struct Stone {
    x: f32,
    y: f32,
    x_offset: f32,
    y_offset: f32,
    rotation: f32,
    x_velocity: f32,
    y_velocity: f32,
    rot_velocity: f32,
    cycles: u32,
}
```

We need to add these to the Stone::new() function as well to give them initial values. The update() function will reset the new fields to random values whenever cycles reaches 0, so we start it at 0 and let update() worry about randomizing the others.

```
impl Stone {
    fn new(x: f32, y: f32) -> Self {
        let x_offset = 0.0;
        let y_offset = 0.0;
        let rotation = 0.0;
        let x_velocity = 0.0;
        let y_velocity = 0.0;
        let rot_velocity = 0.0;
        let cycles = 0;
        Stone {
            x,
            y,
            x_offset,
            y_offset,
            rotation,
            x_velocity,
            y_velocity,
            rot_velocity,
            cycles,
        }
    }
}
```

We don't need to change the Model struct at all since it just keeps a vector of Stones. But we do need to remove the set_loop_mode() call from the beginning of model() since we want the main loop to always iterate for an animation.

The main changes to implement the animation are to the update() function. We keep the main for loop ```for stone in &mut model.gravel```, but only want to randomize the stone when stone.cycles is 0. Rather than setting the offsets and rotation directly, we compute a new random "target": new_x, new_y, and new_rot, as well as a random number of cycles to reach it: new_cycles. We then compute the velocity values by subtracting the current from the new and dividing by the number of cycles.

If stone.cycles is greater than 0, we just add the velocity values to the current values to move the square towards its target and decrement cycles by 1. Here is the updated update() function:

```
fn update(_app: &App, model: &mut Model, _update: Update) {
    let mut rng = StdRng::seed_from_u64(model.random_seed);
    for stone in &mut model.gravel {
        if stone.cycles == 0 {
            let factor = stone.y / ROWS as f32;
            let disp_factor = factor * model.disp_adj;
            let rot_factor = factor * model.rot_adj;
            let new_x = disp_factor * rng.gen_range(-0.5, 0.5);
            let new_y = disp_factor * rng.gen_range(-0.5, 0.5);
            let new_rot = rot_factor * rng.gen_range(-PI / 4.0, PI / 4.0);
            let new_cycles = rng.gen_range(50, 300);
            stone.x_velocity = (new_x - stone.x_offset) / new_cycles as f32;
            stone.y_velocity = (new_y - stone.y_offset) / new_cycles as f32;
            stone.rot_velocity = (new_rot - stone.rotation) / new_cycles as f32;
            stone.cycles = new_cycles;
        } else {
            stone.x_offset += stone.x_velocity;
            stone.y_offset += stone.y_velocity;
            stone.rotation += stone.rot_velocity;
            stone.cycles -= 1;            
        }
    }
}
```

When we compile and run this, the squares indeed move around gradually, so it looks like we've done something right. But the results don't look very random!

![](images/schotter4a.png)

When I first coded this, I puzzled for awhile why it didn't produce random results. So kudos if you see the problem! Clicking Randomize makes the squares move to a new configuration, but still not very random. The problem is subtle. Remember back in schotter2 when we added a seeded random number generator to give consistent results each time through the loop? We still start each iteration of update() with the same seed, so get the same sequence of random numbers each time. Now that we are animating it, we don't want to repeat the random numbers.

One way to fix this is to add rng, our seeded random number generator, to the model, initializing it with the seed in the model() function, and reinitializing it whenever we press 'R' or click Randomize. But it is probably easier just to abandon the seeded random number generator and use the Nannou random_range() function to generate them. Let's try that, replacing the four occurrences of ```rng.gen_range()``` with ```random_range()```:

```
  let new_x = disp_factor * random_range(-0.5, 0.5);
  let new_y = disp_factor * random_range(-0.5, 0.5);
  let new_rot = rot_factor * random_range(-PI / 4.0, PI / 4.0);
  let new_cycles = random_range(50, 300);
```

Now compiling will give some warnings about the unused library Rng and the unused variable rng, but it does appear to work like we expect. The squares constantly rotate and move around, forming continually changing random patterns. It's fun to watch. Changing the sliders doesn't have an immediate effect like before; squares still move to their original targets. But the new targets will be controlled by the new settings, so after a few seconds the effects of the changes will be seen.

Of course, the Randomize button now has no effect at all since we aren't using the seeded random number generator. The question is whether we should continue using random_range and clean up the rng code, or switch back to using the seeded random number generator, but in a more effective way. Let's imagine that we did switch back so clicking Randomize changed the stream of random numbers. How would the result change?

Well, when we click Randomize the target square positions would be different from the positions if we hadn't clicked Randomize. But they are still random, so it isn't a difference we would be able to see. Put another way, if we just looked at the output window and someone else controlled the control panel without us knowing what they did, we wouldn't be able to tell when they click Randomize. Changing the sliders would have a noticable effect after a few seconds, but not clicking Randomize.

So we no longer need the seeded RNG or the Randomize button. So let's clean up our code. Starting at the top, we first delete the two ```use nannou::rand``` lines since we no longer need to use that library. Then we delete the randomize, seed_label, and seed_text widgets from the widget_ids! macro. Struct Stone doesn't change, but we can delete random_seed from the model and the model() function (in two places). In update() we delete the ```let mut rng =``` line. In key_pressed(), we delete the ```Key::R``` block. Finally, in ui_event() we delete the Randomize button, Seed label, and Seed text codes.

Now compiling gives no warnings, and running the program works as we envisioned. Let's make some improvements. Right now, all of the squares (except for the top row) are in constant motion. Let's calm it down a bit by making the squares rest occasionally. We can stop a square from moving by setting the velocity values to 0. We'll do this randomly by adding the following code:

```
if stone.cycles == 0 {
    if random() {
        stone.x_velocity = 0.0;
        stone.y_velocity = 0.0;
        stone.rot_velocity = 0.0;
        stone.cycles = random_range(50, 300);
    } else {
      // generate new velocity values as before
```

A simple change, but I think it looks nicer when not all of the squares are moving at once. Of course, you may have a different opinion! And my opinion will change from time to time. So let's add a control to specify how often squares will be still. Instead of using random(), which just randomly returns true or false with a 50% change, we'll use random_f32(), which returns a random value between 0 and 1, and compare that to the probability we will set with a slider.

First, we need to add a variable for this to Model. We'll call it "motion", and add it to the struct Model (I put ```motion: f32,``` between rot_adj and gravel), add a line ```let motion = 0.5;``` to the model() function, and include ```motion,``` in the correct place in our assignment to the_model. Then in update() we change ```if random()``` to ```if random_f32() > model.motion```.

Now we add the slider. There is conveniently a space in the control panel since we deleted the Randomize button. Following the pattern we used in schotter3, we find widget_ids! and add ```motion_label``` and ```motion_slider``` to it. Then we add code to create the label and slider to ui_event():

```
// Motion label
widget::Text::new("Motion")
    .down_from(model.ids.rot_label, 10.00)
    .w_h(125.0, 30.0)
    .set(model.ids.rot_label, ui);

// Motion slider
for value in widget::Slider::new(model.motion, 0.0, 1.0)
    .right_from(model.ids.motion_label, 10.0)
    .w_h(150.0, 30.0)
    .label(&model.rot_adj.to_string())
    .set(model.ids.rot_slider, ui)
{
    model.motion = value;
}
```

You can, of course, just copy and paste this code into the function. What I actually did to create it was to copy the rotation label and slider code and paste a copy at the end of the function, then changed it to implement the new slider. Note especially that the range for the Motion slider is 0 to 1, since that is what random_f32() will return. We set the default value of 0.5 in the model() function.

Now we can adjust the behavior to our liking. Setting Motion to 0 (no motion) stops all of the squares (after finishing their current cycle). Setting it to 1 (full motion) never stops squares from moving. If it is set a bit more than 0, it will be mostly still but with occasional movement. Playing with the controls, we can get some interesting effects. For example, I set Displacement and Rotation to 0 and waited for all the squares to go back to their home positions. Then I set Motion to 0 to stop everything and set Displacement and Rotation back to 1. Then I increased Motion to about 0.2 and waited a bit while some of the squares moved, and set it back to 0 to freeze it before some of the squares moved from their original positions, giving the following (which would not be possible with earlier schotter versions):

![](images/schotter4b.png)