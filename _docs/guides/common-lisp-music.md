---
title: "\"Common Lisp Music\"-style example"
---

Here's an example of how to do
[CLM](https://ccrma.stanford.edu/software/clm/)-style musicmaking in Extempore.
First, we load in the DSP library:

~~~~ xtlang
(sys:load "libs/core/audio_dsp.xtm")
~~~~

We need a *master* DSP callback. We can call it whatever we like, but by
convention we call it `dsp`. It's type **must** be
`[float,float,i64,i64,float*]*`

~~~~ xtlang
(bind-func dsp
  (let ((oscil (osc_c 0.0))) ;; instantiates an oscilator with phase 0.0
    (lambda (in:float time:i64 chan:i64 dat:float*)
      (if (= chan 0)         ;; if sample request is for channel 0 (i.e. left)
          (oscil 0.3 220.0)  ;; return sound a 220hz tone with amplitude 0.3
          0.0))))            ;; otherwise all other channels return silence
~~~~

We need to tell extempore what we called our DSP callback function using
`dsp:set!`. We only call `dsp:set!` once per session---from that point until the
end of the session this function is the audio callback

~~~~ xtlang
(dsp:set! dsp)
~~~~

We can of course recompile `dsp` on-the-fly:

~~~~ xtlang
(bind-func dsp
  (let ((oscil (osc_c 0.0)))
    (lambda (in:float time:i64 chan:i64 dat:float*)
      (if (= chan 1)         ;; switch from left channel to right channel
          (oscil 0.3 330.0)  ;; change to 330.0 hz
          0.0))))
~~~~

Now we have two oscillators, one for the left channel and one for the right. We
also introduce `DSP`, a type alias for `[float,float,i64,i64,float*]*`

~~~~ xtlang
(bind-func dsp:DSP
  (let ((oscil_left (osc_c 0.0))
        (oscil_right (osc_c 0.0)))
    (lambda (in time chan dat)
      (cond ((= chan 0) ;; left
             (oscil_left 0.3 330.0))
            ((= chan 1) ;; right
             (oscil_right 0.3 220.0))
            (else ;; any other channels
             0.0)))))
~~~~

Equivalently, Extempore's audio DSP library has multi-channel oscillators, so we
can clean up our code a bit:

~~~~ xtlang
(bind-func dsp:DSP
  (let ((oscil (osc_mc_c 0.0)))
    (lambda (in time chan dat)
      (oscil chan 0.3 220.0)))) ;; any number of channels
~~~~

Let's add a frequency sweep. `SRf` is a global variable which holds the audio
sample rate (f for `float`)

~~~~ xtlang
(bind-func dsp:DSP
  (let ((oscil (osc_mc_c 0.0))
        (duration (* SRf 1.0))      ;; samplerate * 1.0 seconds
        (range (/ 440.0 duration))) ;; rise up to 440.0 hz
    (lambda (in time chan dat)
      ;; explicit conversion required to coerce time (i64) into a float
      (oscil chan 0.3 (* range (% (i64tof time) duration))))))
~~~~

Now, we can "granulate" the rising oscillator. With the default settings, the
the granulator will have little effect on the original signal.

~~~~ xtlang
(bind-func dsp:DSP
  (let ((oscil (osc_mc_c 0.0))
        (duration (* SRf 1.0))      ;; samplerate * 1.0 seconds
        (range (/ 440.0 duration))
        (grains (granulator_c 2)))  ;; setup granulator for two channels
    (lambda (in time chan dat)
      (grains chan time  ;; granulator takes chan, time and input
              (oscil chan 0.7 (* range (% (convert time) duration)))))))
~~~~

Now the fun part---start playing around with the granulator's settings!
Extempore's granulator is stochastic and supports lo and hi ranges for most
parameters, making a stochastic choice for each grain in that range. You can get
determinstic behaviour by making lo and hi the same value.

~~~~ xtlang
(bind-func dsp:DSP
  (let ((oscil (osc_mc_c 0.0))
        (duration (* SRf 1.0))
        (range (/ 440.0 duration))
        (grains (granulator_c 2)))
    ;; set some initial values
    (grains.iot 500)     ;; inter-offset time (time between grains in samples)
    (grains.dlo 1000.0)  ;; shortest (i.e. low)  duration (in samples)
    (grains.dhi 10000.0) ;; longest  (i.e. high) duration  (in samples)
    (grains.rlo 0.5)     ;; slowest  (low) playback rate  (%50)
    (grains.rhi 2.0)     ;; fastest  (high) playback rate (%200)
    (lambda (in time chan dat)
      (grains chan time  ;; granulator takes chan, time and input
              (oscil chan 0.7 (* range (% (convert time) duration)))))))
~~~~
