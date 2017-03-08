# Part 15: Input



## Input system

* Abstract over various input devices
    * Keyboard
    * Gamepad
    * Touch
    * Pen
    * Synergy
* `input_controller.h`
    * Controller is a collection of named *buttons* and *axes*
    * Buttons provide a float value and an up/down state
    * Axes provide a Vector3 value
    * Connected/disconnected state
    * Rumble (if supported)
    * `windows_wndproc_mouse.h` `windows_wndproc_mouse.cpp`
* `input_manager.h`
    * Keeps track of all controllers
    * Called each frame to poll controllers
    * On windows, `Application` feeds events from `Window` into `InputManager`



## Events vs state

* Main system design is state based (mouse down?)
* This can cause rapid events to be missed
    * Mouse up, down, up in same frame
    * Especially a problem when frame times are long
* `RawInputQueue` -- alternative `input_manager.h`
    * Queue of all input events
    * Still not perfect, because controllers are polled in order
    * Doesn't know order of buttons pressed between controllers
* Continue to use this hybrid interface or switch to just event based?



## Text input

* `input_controller.h`
* `KeyboardInputController::keystrokes()`
    * List of characters and special keypresses (tab, delete, etc) for text editing
    * Characters are listed as UTF-8 strings
    * Special keypresses as enums
