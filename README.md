Go to [this link](https://gitpitch.com/hoj201/stater_presentation)

## Contents
- what is the point of this talk?
- what is the challenge?
  - i2c points -> postures
  - reality -> i2c stream is many->one
  - we can't use too much processing (battery/cpu limits)
  - we can't integrate acceleration to get position/velocity
- intro to machine learning
  - machine learning as curve fitting (or pdf fitting)
  - the challenge
  - overfit
- what is stater?
  - lift plugin
  - kyu plugin
  - walk plugin
  - recal plugin


## What is the point of this talk?

By the end of this talk you should:
  - understand what decides when the device buzz.
  - understand what areas are not taken into account in this decision.
  - understand a few limits on what is possible.

Important, because you only have one machine learning engineer at the moment. Knowing these things will help you help me.


## What is the challenge?

The input looks like this
```json
{"side":{"accelerometer":{"x":-0.9215088,"y":0.010009766,"z":0.019165039},"quaternion":{"w":-0.36803147,"x":0.5970288,"y":-0.347097,"z":-0.6226018},"gyro":{"x":-1.7089844,"y":-0.24414062,"z":0.24414062},"temperature":36,"altitude":-2961,"button":0},"time":1495047945627}
{"side":{"accelerometer":{"x":-0.90405273,"y":-0.1060791,"z":0.033447266},"quaternion":{"w":-0.36077243,"x":0.6014298,"y":-0.33910534,"z":-0.6270031},"gyro":{"x":-1.4038086,"y":0.18310547,"z":0.24414062},"temperature":36,"altitude":-2964,"button":0},"time":1495047945653}
{"side":{"accelerometer":{"x":-0.9477539,"y":-0.20654297,"z":0.08959961},"quaternion":{"w":-0.3563688,"x":0.6040387,"y":-0.3349464,"z":-0.62924516},"gyro":{"x":-0.4272461,"y":0,"z":0.5493164},"temperature":36,"altitude":-2961,"button":0},"time":1495047945693}
{"side":{"accelerometer":{"x":-0.99157715,"y":-0.22045898,"z":0.1048584},"quaternion":{"w":-0.35570565,"x":0.60106194,"y":-0.334771,"z":-0.6325555},"gyro":{"x":0.12207031,"y":-0.61035156,"z":0.91552734},"temperature":36,"altitude":-2956,"button":0},"time":1495047945733}
{"side":{"accelerometer":{"x":-0.94470215,"y":-0.14831543,"z":0.12158203},"quaternion":{"w":-0.35564506,"x":0.5939218,"y":-0.3380673,"z":-0.6375611},"gyro":{"x":0.5493164,"y":-0.79345703,"z":0.9765625},"temperature":36,"altitude":-2953,"button":0},"time":1495047945773}
{"side":{"accelerometer":{"x":-0.8881836,"y":-0.046020508,"z":0.11413574},"quaternion":{"w":-0.35887003,"x":0.5844455,"y":-0.34159788,"z":-0.6426093},"gyro":{"x":0.79345703,"y":-1.0986328,"z":0.36621094},"temperature":36,"altitude":-2956,"button":0},"time":1495047945815}
```
The output we want looks like this
```json
{"action":"lift.safe","body":{"backAngle":71.04457255759334,"end":1495048190609,"start":1495048188889,"twistAmount":17.977664314273042,"type":"none"}}
{"action":"lift.risky","body":{"backAngle":68.45104241395217,"end":1495048202440,"start":1495048200682,"twistAmount":50.487119749529256,"type":"twist"}}
{"action":"lift.safe","body":{"backAngle":59.843594470741394,"end":1495048203361,"start":1495048202561,"twistAmount":15.918158662410605,"type":"none"}}
```

### Challenge 1: Sensor readings are many to one
We want a triangle to commute. (universe, labels, sensor outputs)

*Example (1,2,3,4,5).  The real labels are determined by the function (x<4), the sensor just tells us if the number is even/odd.*

- Relevant example: Things that are not risky lifts can produce i2c streams that are identical to those of risky lifts.

### Challenge 2: Computational power is finite
We have impressive hardware, but it's not infinite.
<img src="https://microship.com/wp-content/uploads/2014/03/Byte-September-1981-AI-cover.jpg">

 - Too much math drains the battery and we only have 40ms to process each point.
 - Too many log messages drains the battery.
 - Impact: Neural nets can't be too deep.
 - Impact: Features must be computable.

### Challenge 3: Our sensors are not perfect
 - Impact: we must filter.
 - Impact: we can't extract position / velocity from acceleration (draw this)

## Machine learning crash course
Example: linear regression.
You have labelled data {(x1,y1), (x2,y2), ...}.  We'd like to find a map which takes xs->ys which is "consistent" with the data.

### Cross validation
 - split the data into a training and a testing set
 - fit a function to the training set
 - test your function on the test set

Do this over and over until you get a function you like.

## Stater

### The lift plugin

 - "sagittal angle".

#### window generation
 we wait for $theta$ to exceed 20 degrees.  Then we wait for it to fall back down.  That defines a window.

### window processing
 We extract a bunch of features like:
  - max saggittal angle during window
  - max twist during the window
  - change in height during the window
  - ...

Then we predict if a squat occured using the height change.
If a squat is predicted, the device will not buzz.

Also, if the sagittal angle is ridic (i.e. > 75 degrees) we will not buzz.

Then we estimate the back angle using the sagittal angle via a linear equation and buzz if the estimated back angle is beyond 72 degrees. With respect to sagittal angle, this means if the sagittal angle is above 36, the device will detect a risky bend, and buzz.

If the back angle is between 31 and 36, then the features are used to predict a twist, and if a twist is detected the device will buzz.

### Calibration
If the device is put on correctly, then the kyu plugin does the work... math.

$$ s = \sum_{i=1}^N \frac{1}{\sigma_i + \epsilon} g_i$$

$$ g_cal = s / |s|$

Otherwise, we use a walk detector and do similar math (i.e. compute the same equation while walking)


### Walk plugin
Processes on 6 second windows.  Computes the ACF function. then does a vanilla NN on top, every second.

$$ acf[f] := \sum_k x[k]x[k+i]$$
