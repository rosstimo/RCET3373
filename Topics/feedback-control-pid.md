<a id="top"></a>

# RCET 3373 — Feedback Control and PID Foundations

*Self-learning guide*

[Topics index](README.md)

<a id="contents"></a>
## Contents

- [1. Why this matters](#why-this-matters)
- [2. What you should be able to do](#learning-outcomes)
- [3. Prerequisites and related topics](#prerequisites)
- [4. Core model and vocabulary](#core-model)
- [5. How it works](#how-it-works)
- [6. Worked examples](#worked-examples)
- [7. Apply, verify, and troubleshoot](#apply-verify-troubleshoot)
- [8. Practice](#practice)
- [9. Answer key](#answer-key)
- [10. What you should be able to explain without notes](#retrieval-check)
- [11. References](#references)

[Back to top](#top) · [Topics index](README.md)

<a id="why-this-matters"></a>
## 1. Why this matters

A feedback controller uses measurements of a system to decide how an actuator should respond.

Robotics is full of these loops:

- motor speed;
- position;
- steering;
- temperature;
- pressure;
- balance;
- battery/current regulation;
- line following.

The embedded-system version of a control loop combines topics already used in this course:

```text
sensor
-> ADC / input measurement
-> software estimate
-> control calculation
-> PWM / output command
-> actuator / plant
-> new sensor measurement
```

The difficult part is not merely writing the letters P, I, and D in code. The loop must have suitable sensing, timing, scaling, actuator limits, and verification.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- identify setpoint, process variable, error, controller output, actuator, and plant;
- explain negative feedback;
- explain proportional, integral, and derivative terms conceptually;
- identify the effect of sample interval on a digital controller;
- explain output saturation and integral windup;
- explain why derivative action is sensitive to measurement noise;
- distinguish steady-state error from transient response;
- identify practical embedded implementation limits;
- propose a measurement-driven tuning/troubleshooting process.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [PIC16F883 ADC and Sensor Conditioning](pic16f883-adc.md#core-model);
- [PIC16F883 Timers](timers.md#core-model);
- [Timing Measurement](measurement-c-timing.md#prediction-evidence).

Related topics:

- later PWM/actuator implementation;
- [Embedded-System Integration and Troubleshooting](embedded-system-integration-troubleshooting.md#timing-budget).

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="feedback-loop"></a>
### Closed-loop model

A basic negative-feedback loop is:

```text
setpoint
   |
   v
[ compare ] -> error -> controller -> actuator -> plant -> measured output
    ^                                                   |
    |___________________________________________________|
                         feedback
```

Definitions:

| Term | Meaning |
| --- | --- |
| Setpoint | desired value |
| Process variable | measured actual value |
| Error | setpoint - measured value |
| Controller output | command calculated from error/history |
| Actuator | hardware that changes the physical system |
| Plant | physical process being controlled |
| Feedback | measured output returned to the controller |

Negative feedback attempts to reduce error.

<a id="sampled-control"></a>
### Digital control is sampled

A microcontroller does not continuously recalculate the controller.

It repeats a loop such as:

```text
wait for sample time
-> read sensor
-> calculate error
-> calculate control output
-> apply output
-> repeat
```

The sample interval is part of the control design.

If the interval changes unpredictably, the controller can behave differently even when the code formula is unchanged.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="proportional"></a>
### Proportional action

Proportional control reacts to current error:

```text
P = Kp × error
```

Larger error produces a larger corrective command.

Increasing `Kp` generally makes the controller respond more strongly, but too much gain can create overshoot or oscillation.

Proportional control alone can leave steady-state error in systems that require a persistent nonzero command to hold the target.

<a id="integral"></a>
### Integral action

Integral action accumulates error over time.

Conceptually:

```text
integral += error × sample_time
I = Ki × integral
```

It can eliminate persistent steady-state error because even a small continuing error keeps building corrective action.

But accumulation creates a major practical problem when the actuator is already saturated.

<a id="derivative"></a>
### Derivative action

Derivative action responds to how quickly error is changing.

Conceptually:

```text
D = Kd × (error - previous_error) / sample_time
```

It can add damping by reacting to rapid change.

However, differentiation also amplifies high-frequency measurement noise. Real implementations often filter or otherwise limit derivative behavior.

<a id="pid"></a>
### PID combination

A conceptual digital PID controller is:

```text
output = P + I + D
```

or:

```text
output =
    Kp × error
  + Ki × accumulated_error
  + Kd × rate_of_error_change
```

The exact discrete-time implementation matters. Different libraries may scale `Ki` and `Kd` differently with sample time.

Do not copy gain values from another implementation without understanding its equation and update interval.

<a id="saturation-windup"></a>
### Saturation and integral windup

Real actuators have limits.

Examples:

```text
PWM duty: 0% to 100%
motor voltage/current limits
valve position limits
heater fully off to fully on
```

If the calculated controller output exceeds the physical limit, the actuator saturates.

If integral error continues accumulating while saturated, the integrator can become much larger than useful. When the system finally returns toward the target, that stored integral can cause long overshoot/recovery.

This is **integral windup**.

Common anti-windup ideas include:

- stop/clamp integral accumulation under defined saturation conditions;
- back-calculate the integrator from the saturated output;
- limit the integral term.

<a id="sampling"></a>
### Sample interval is an engineering parameter

A sample interval should be:

- fast enough to observe/control relevant plant behavior;
- slow enough to allow reliable sensing/computation;
- consistent enough that the discrete controller equation remains meaningful.

A timer-derived periodic event is usually a better timebase than an arbitrary blocking delay.

The appropriate sample rate depends on the physical system. Faster is not automatically better.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="p-example"></a>
### Worked example: proportional controller

Suppose:

```text
setpoint = 100
measured = 92
Kp = 2
```

Error:

```text
error = 100 - 92 = 8
```

Proportional term:

```text
P = 2 × 8 = 16
```

If the actuator command is allowed to increase by 16 units, the controller responds in the direction that should reduce the error.

<a id="sample-time-example"></a>
### Worked example: integral accumulation depends on time

Suppose:

```text
error = 4
sample time = 0.020 s
```

One integration step adds:

```text
error × sample_time
= 4 × 0.020
= 0.08 error·s
```

If another implementation updates every 0.100 s, the same error contributes 0.4 error·s per update.

Therefore sample interval belongs in the control equation.

<a id="saturation-example"></a>
### Worked example: actuator saturation

Suppose the controller calculates:

```text
command = 135% PWM
```

but the actuator can only accept:

```text
0% to 100%
```

The physical output is limited to 100%.

The controller should recognize that saturation exists. Continuing to integrate a large positive error indefinitely can create windup.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="control-checklist"></a>
### Embedded control checklist

Before tuning gains, verify:

1. sensor direction and calibration;
2. setpoint units;
3. actuator direction;
4. actuator limits;
5. sample interval and jitter;
6. timer/interrupt behavior;
7. ADC scaling and noise;
8. output scaling/PWM mapping;
9. sign of the feedback;
10. safe bounds for the physical system.

If increasing the command moves the measured variable in the wrong direction relative to the control law, fix that sign/plant understanding before tuning.

<a id="tuning-workflow"></a>
### Measurement-driven tuning workflow

A simple educational process is:

1. log setpoint, measurement, error, and output;
2. start with conservative gains;
3. change one design/tuning variable at a time;
4. observe rise time, overshoot, oscillation, steady-state error, and settling;
5. verify actuator saturation;
6. inspect whether integral windup occurs;
7. inspect derivative noise sensitivity;
8. repeat using measured evidence.

A controller that “feels better” without recorded evidence is difficult to compare or reproduce.

<a id="implementation-limits"></a>
### Microcontroller implementation limits

An 8-bit MCU introduces practical concerns:

- limited numeric range;
- integer/fixed-point scaling;
- overflow;
- ADC resolution/noise;
- PWM resolution;
- sample timing;
- interrupt latency;
- CPU time;
- saturation limits.

These constraints do not invalidate PID. They become part of the implementation design.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. Define setpoint, process variable, and error.
2. What does negative feedback attempt to do?
3. What does proportional action respond to?
4. What problem can integral action solve?
5. What problem can integral action create when the actuator saturates?
6. Why can derivative action react strongly to sensor noise?
7. Why does sample time appear in the integral and derivative terms?
8. A controller calculates 120% duty cycle but the actuator is limited to 100%. What has happened?
9. Why should sensor and actuator direction be verified before tuning gains?
10. Name four quantities you would log during tuning.
11. Why is a timer-derived periodic sample usually preferable to an arbitrary blocking delay?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. Setpoint is desired value, process variable is measured actual value, error is their difference according to the chosen sign convention.
2. Reduce the difference between desired and measured behavior.
3. Current error.
4. Persistent steady-state error.
5. Integral windup and excessive overshoot/recovery.
6. Differentiation emphasizes rapid changes, including high-frequency measurement noise.
7. A discrete controller approximates time-domain integration/differentiation through sampled updates; changing the interval changes those operations.
8. Output saturation.
9. A wrong sign can create positive feedback, driving the system away from the target.
10. Examples: setpoint, measured value, error, controller output, P/I/D terms, sample time.
11. A hardware timebase makes the control update interval explicit and more stable while allowing the CPU to do other work.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- draw a closed feedback loop;
- define setpoint, measurement, error, controller, actuator, and plant;
- explain P, I, and D contributions;
- explain why sample time matters;
- explain saturation and integral windup;
- explain derivative noise sensitivity;
- identify embedded implementation constraints;
- outline a safe, evidence-based tuning process.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *Software PID Control of an Inverted Pendulum Using the PIC16F684*, AN964 — https://www.microchip.com/en-us/application-notes/an964
  - Used for: embedded PID structure, sampled implementation, fixed-resource PIC control example, and practical closed-loop context.

- Microchip Technology Inc., *Digital Signal Processing with the PIC16C74*, AN616 — https://www.microchip.com/en-us/application-notes/an616
  - Used for: supplemental PIC16 digital-control/PID implementation context.

- Microchip Technology Inc., *PIC MCU CCP and ECCP Tips 'n Tricks*, DS41214B — https://ww1.microchip.com/downloads/en/devicedoc/41214b.pdf
  - Used for: supplemental PWM/actuator-output context. Verify actual PIC16F883 CCP features in its data sheet before implementation.

[Back to top](#top) · [Topics index](README.md)
