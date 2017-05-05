# Overview

*Interaction* is a graphical user interface library for embedded systems using [eGFX](https://github.com/ehughes/eGFX). Examples of *Interaction* in action can be found on NXP's LPCXpresso 54608 dev board powering my [senior design project](https://github.com/MattPennock/Embedded-Audio-Mixer).

![Embedded Audio Mixer](/Images/LPC.jpg) 

Currently there is support for Buttons and Sliders. Check out my [YouTube Demo](https://youtu.be/au6VgSrRr1A) for the library in action.

The *Interaction* touch library abstracts touch interactions for sliders and buttons into three stages:

* Static Initilization
* Draw Commands to eGFX's frame buffer
* Updates to Global State on Touch Commands

# Example Application Code

After initilizing all interaction structures to be used within the GUI, draw commands and state updates are designed to be modular for use within a main loop or interrupts. An interrupt based use case is shown below:

## Initilization

Start by statically allocating global varibales such as volume and a play/pause button to be updated in application code. A function `GUI_INIT` is written to be called within `main` to handle inital setup.

```c
#define BUTTON_HEIGHT 32
#define BUTTON_WIDTH  32
#define SLIDER_HEIGH  32
#define SLIDER_WIDTH 200

float g_Volume;
int   g_VolumeX;
bool  g_Play;
Button_t PlayButton;
Slider_t VolumeSlider;

void GUI_Init(void)
{
    eGFX_Box Bounds;

    Set_Box(&Bounds, 32, 32, BUTTON_HEIGHT, BUTTON_WIDTH);
    Init_Button(&PlayButton, Bounds, k_Green, "Play", &g_play, k_PlayGroup);

    Set_Box(&Bounds, 32, 256, SLIDER_HEIGHT, SLIDER_WIDTH);
    Init_Slider(&VolumeSlider, Bounds, k_Cyan, "Volume", &g_volume, 0, 63, &g_VolumeX);
}
```

## Update

After a touch event is received over I2C the x and y pixel coordinates are used to check if GUI elements have been interacted with. Member functions `Detect_Touch()` and `Update_Button()`/`Update_Slider()` work in tandem.

Our demonstration code will be in a fictional interrupt `TouchIRQHandler()`, however polling within a main loop also works.

```c
void TouchIRQHandler()
{
    /* Code to check Interrupt Mask & clear Register omitted */

    if( Detect_Touch(&PlayButton->Box, x, y) )
        Update_Button(&PlayButton);

    if( Detect_Touch(&VolumeSlider->Box, x, y) )
        Update_Slider(&VolumeSlider, x);
}
```

If the Play button was pressed, the global flag `g_Volume` will be asserted true. For Team ECHO's *Embedded Audio Mixer* a state machine called after GUI Updates will check global variables and modify operation.

## Draw Commands

After the button & sliders have been updated, make calls to change their state. The Interaction library is tightly coupled to Eli Hughe's implementation of eGFX avaliable here: [TODO: LINK TO eGFX]

```c
Draw_Button(&PlayButton);
Draw_Slider(&VolumeSlider);
```
# Avaliable Types

## enum **Color_Wheel**

Colors encoded in 5:6:5 RGB `uint16_t` format. Used with ARM's LCD Controller IP Block. A pixel searializer handles conversion to other display formats on TFT panels.

![Color Wheel](/Images/Color.jpeg)

### C Decleration:
```c
typedef enum
{
	k_Black         = 0x0000u,
	k_White         = 0xFFFFu,
	k_Yellow        = 0x07DFu,
	k_YellowOrange  = 0x067Fu,
	k_Orange        = 0x053Fu,
	k_RedOrange     = 0x03FFu,
	k_Red           = 0x011Fu,
	k_RedPurple     = 0x913Eu,
	k_Purple        = 0xB911u,
        k_Blue          = 0xB920u,
	k_LightBlue     = 0xC3C0u,
	k_Cyan          = 0xD580u,
	k_Green         = 0x05C0u,
	k_Lime          = 0x0670u,
	k_HotPink       = 0xED9Fu,
	k_Gray          = 0xC618u
} Color_Wheel;
```

## struct **Button_t**
---

Button's control a global boolean variable `Flag`, all other member variables are encapsulations for drawing the structure.

![Button](/Images/Button.PNG)

### C Decleration:
```c
typedef struct {
    eGFX_Box        Box;            /* bounds */
    Color_Wheel     Color;          /* color */
    char*           Label;          /* text written within the button */
    bool            isFilled;       /* active button should be filled if true */
    bool*           Flag;           /* memory address of global variable under control */
    uint32_t        Group;          /* useful for removing fill from inactive buttons */
} Button_t;
```

## struct **Slider_t**
---

Slider's apply linear control to a global float. The `Control` member variable will be mapped to the current x position of user's touch input when a Slider is pressed. The `lowerBound`, `upperBound` and `scaleFactor` members control the range of the float.

![slider](/Images/slider.PNG)

### C Decleration:

```c
typedef struct {
    eGFX_Box        Box;            /* bounds */
    Color_Wheel     Color;          /* color */
    char*           Label;          /* label of slider written to the left */
    int             *xPosition;     /* memory address of slider's active area's  */
    float           *Control;       /* memory address of global float under control, written to right of slider */
    float           lowerBound;     /* least value assigned to Control */
    float           upperBound;     /* greatest value avaliable to Control */
    float           scaleFactor;    /* internal variable to linearlly interpolate between lower and upper bounds */
} Slider_t;
```

# Member Functions

## **Init_Button(...)**

Initilize the `Button_t` structure provided. Must be called before Update and Draw operations.

### C Decleration:
```c
void Init_Button(Button_t *B, 
                 eGFX_Box Box,
                 Color_Wheel color, 
                 char* label, 
                 bool* flag,
                 Button_Group group);
```


## **Draw_Button(...)**

Draw the button based on current state to the eGFX background frame buffer. Requires `#include eGFX_Driver.h` for the memory address.

### C Decleration:
```c
void Draw_Button(Button_t *B);
```

## **Update_Button(...)**

Set associated global variable `Control` to `true`; The application code is responsible for setting the variable back to `false` and update behavior.

### C Decleration:
```c
void Update_Button(Button_t *B);

## **Init_Slider(...)**
Initilize the `Slider_t` structure provided. Must be called before Update and Draw operations.

### C Decleration:
```c
void Init_Slider(Slider_t *S,
                 eGFX_Box Box,
                 Color_Wheel color,
                 char* label,
                 float* control,
                 float lower_bound,
                 float upper_bound,
                 int* xPosition);

```

## **Draw_Slider(...)**

Draw the slider based on current state to the eGFX background frame buffer. Requires `#include eGFX_Driver.h` for the memory address.

### C Decleration:
```c
void Draw_Slider(Slider_t *S);
```

## **Update_Slider(...)**
Change the x coordinates of the active & inactive regions to be drawn & update the global float pointed to by `Control`. 

### C Decleration:
```c
void Update_Slider(Slider_t *S, int x);
```

## **Detect_Touch(...)**

Return true if point (x,y) is within the bounding box of a GUI element.

### C Decleration:
```c
bool Detect_Touch(eGFX_Box *Box, int x, int y);
```

## **Set_Box(...)**

Modify an eGFX_Box with upper left (x,y) coordinate and width/height.

### C Decleration:
```c
 void Set_Box(eGFX_Box *Box, 
              int x, int y, 
              int width, int height);
```
