# simple-psychopy-visual-task
A simple client server configuration for delivering multimonitor stimuli.
Intended to be used in conjunction with the 
[TreadmillIO Project](https://github.com/kemerelab/TreadmillIO).



## StateMachine or VisualStimulus Tasks
The file `VisualStimulusExperiment.py` executes a simple PsychoPy stimulus set that listens on a ZMQ
socket for commands. The example TreadmillIO configuration file below executes a simple statemachine
that interacts with the PsychoPy process. The key items are the `VisualCommsPort` setting in the
`Preferences`, and the `Visualization` state type. 

Following is an example YAML configuration file that in conjunction with the `VisualStimulusExperiment.py`
code  creates a grey screen and then presents a flashing checkerboard randomly either in the left or right 
visual hemifields (calibrated for a large 40" curved display). 


```yaml
Info: # "Info" configuration options can be overridden by command line options
  MouseID: ExampleMouseName # Used to name saved data file
  TaskType: Visual Classical Conditioning

Preferences:
  HeartBeat: 1000 # The interval at which a heart-beat info message is printed to screen (in ms) 
  AudioFileDirectory: '/path/to/simple-sound-stimuli/Sounds/48kHz' # Directory in which sound stimulus files are stored
  RandomSeed: 345 # Useful for repeatability
  VisualCommsPort: 5556
  LogCommands: True

GPIO: # A list of GPIO pins used in the task
  Camera: # This is an arbitrary label used later to configure Reward
    Number: 1 # This is the pin number from the board
    Type: 'Input_Pulldown' 
    Power: False
  Lick: # This is an arbitrary label used later to configure Reward
    Number: 2 # This is the pin number from the board
    Type: 'Input_Pulldown' # Pin type is either 'Input' or 'Output'; special cases could be 'Input_Pulldown' or 'Input_Pullup' 
                           # (where the pull-down/pull-up resistors are enabled).
    Power: True # The 'Power' option is for 'Input' type GPIOs. It configures whether the Aux-GPIO is set to high, 
                # enabling the power channel of the audio jack.
  ScreenLeft: # This is an arbitrary label used later to configure Reward
    Number: 3 # This is the pin number from the board
    Type: 'Input_Pulldown' 
    Power: True

  ScreenRight: # This is an arbitrary label used later to configure Reward
    Number: 4 # This is the pin number from the board
    Type: 'Input_Pulldown' 
    Power: True
  Reward:
    Number: 5
    Type: 'Output'
    Mirror: True # The 'Mirror' option is for 'Output' type GPIOs. It configures whether the associated Aux-GPIO pin is raised/lowered 
                 # whenver the GPIO itself is. The Aux-GPIO is connected to an LED, which allows for visualization of the GPIO state.
  GreyScreenIO:
    Number: 6
    Type: 'Output'
    Mirror: True 
  Disable7:
    Number: 7
    Type: 'Input_Disable'
  Disable8:
    Number: 8
    Type: 'Input_Disable'
  Disable9:
    Number: 9
    Type: 'Input_Disable'
  Disable10:
    Number: 10
    Type: 'Input_Disable'
  Disable11:
    Number: 11
    Type: 'Input_Disable'
  Disable12:
    Number: 12
    Type: 'Input_Disable'              

Maze:
  Type: 'StateMachine'

StateMachine: # Note - will start at the state below with the parameter "FirstState: True". Otherwise
              # the first state listed will be first.
  BlankScreen:
    Type: 'Visualization'
    Params:
      VisType: 'Fixed' # Can be either "Fixed" or "Random"
      Command: 'GREY'  # Command string sent to VisualStimulusServer
    NextState: 'Raise'
  Raise: # Raise a GPIO during the grey screen period
    Type: 'SetGPIO'
    Params:
      Pin: 'GreyScreenIO'
      Value: 1
    NextState: 'Intertrial'
  Intertrial:
    Type: 'Delay'
    Params:
      Duration: 'Fixed' # Can be either "Fixed" or "Random"
      Value: 5000
    NextState: 'Lower'
  Lower:
    Type: 'SetGPIO'
    Params:
      Pin: 'Reward'
      Value: 0
    NextState: 'StimulusPresentation'
  StimulusPresentation:
    FirstState: True    # Start statemachine here!s
    Type: 'Visualization'
    Params:
      VisType: 'Random' # Random visualizations are currently uniformly distributed based on the specified
                        # relative probabilities.
      Options:
        Right:
          Command: 'RIGHT' # Command sent to VisualStimulusServer
          Probability: 1   # Relative probabilitity. These will be normalized by the sum of all options.
        Left:
          Command: 'LEFT'
          Probability: 1
    NextState: 'StimulusPresentationDelay'
  StimulusPresentationDelay:
    Type: 'Delay'
    Params:
      Duration: 'Exponential' # Random duration is a bounded exponential.
      Rate: 1000              # Rate of exponential distribution in ms.
      Min: 2000               # Minimum value added to samples from distribution.
      Max: 4000               # All samples are truncated to this value.
    NextState: 'Reward'
  Reward:
      Type: 'Reward'   # We could also trigger reward by using the SetGPIO command. The "Reward" states
                       # are 'non-blocking', meaning the state machine will move along to the next state.
      NextState: 'BlankScreen'
      Params:
        DispensePin: 'Reward' # (Required!) Label of GPIO for output pin which should trigger reward dispensing. 
        PumpRunTime: 100 # How long to run the reward pump for. Note that this is also the duration
                          # of time that any associated Beep will be played.
        RewardSound: 'RewardSound' # The name of the auditory stimulus (of type "Beep") which should be played 
                              # at the time of reward. This plays for the same duration of the pulse which
                              # triggers the pump.

AuditoryStimuli:
  Defaults: # Default values for stimulus parameters
    Filename: '' # Possible to specify a default sound file
    BaselineGain: 0.0 # Volume of stimulus. Corresponds to the peak volume for a 'Localized' stimulus 
    Type: 'Localized' # 'Localized' (landmark), 'Background', or 'Beep'
    Modulation: # Parameters which define how 'Localized' stimuli are spatially-modulated
      Type: Linear # Currently only 'Linear'
      CenterPosition: 0.0 # Position of Localized stimulus in VR space
      Width: 25 # cm in which sound is on (full width)
      #Rate: -20.0 # dB/cm - NOT CURRENTLY IMPLEMENTED BUT WOULD BE NICE FOR THE FUTURE
      CutoffGain: -60.0 #dB at cutoff - The change in gain as a function of position is determined
                        #  by the Width, BaselineGain, and CutoffGain parameters.
                        #  Note that true "Off" corresponds in our system to -90 dB.
                        #  So depending on SNR and perceptual ability, a CutoffGain greater
                        #  than that value might be noticeable.  

  StimuliList:
    BackgroundSound: # This label is abitrary. In the case of "Beep" stimuli, it is used to configure Reward
      Type: 'Background' # Stimuli type - 'Localized', 'Background', or 'Beep'
      BaselineGain: -5.0
      Filename: 'pink_noise.wav'
      Color: 'pink' # Matplotlib color name used for visualization system
    RewardSound:
      Type: 'Beep'
      Filename: 'tone_11kHz.wav'
```


