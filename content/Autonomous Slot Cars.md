The code is [HERE](https://github.com/Pi3dra/QSlotCar)

This project explores the design of an autonomous slot car system using embedded electronics, networking, and reinforcement learning. The goal is to control a slot car autonomously by observing the electrical behavior of the track and learning an optimal driving strategy.

## Early Challenges & Hardware Choices

We quickly encountered the limits of the Raspberry Pi, notably its lack of analog inputs. This led us to integrate an **ADC (ADS1115)** to measure track voltage accurately. Early experiments involved reverse engineering the controller, understanding voltage noise introduced by the trigger, and learning, sometimes accidentally how grounding and shared references are critical when mixing power sources.

After initial struggles trying to do everything on a **single** Raspberry Pi, we moved to a **two-Raspberry-Pi** architecture:

One Pi dedicated to **sensing and serving data** (voltage measurement + web interface)

One Pi dedicated to **control and learning** (Q-learning algorithm and motor control)

The two devices communicate over Ethernet with static IPs, yielding very low latency (~0.6 ms), which is sufficient for real-time control.

## Control Strategy & Learning

The system relies on Q-learning, with discretized voltage levels representing the car’s state. Early versions of the algorithm struggled with:

- Voltage drops in curves
- Low-speed stalling
- Incorrect penalization when the car left the track

These issues were solved by refining the reward function:

- Penalizing non-acceleration actions at low speed
- Penalizing the state that caused a track exit rather than the state measured afterward

Parameter tuning (especially speed step size and number of states) proved critical. A speed increment of 3–5% offered the best trade-off between stability and responsiveness.

## Results & Experiments

On a larger merged track with long straight sections, the system learned to slightly accelerate on straights while remaining cautious in curves. By the end of testing, the autonomous car completed 20 laps in 1 minute 30 seconds, compared to 1 minute 15 seconds for a human driver an encouraging result for a first implementation.

## Lessons Learned

This project evolved from basic electronics experiments into a full cyber-physical system, combining:

- Embedded hardware
- Networked communication
- Web-based monitoring
- Reinforcement learning

Most progress came not from following a perfect plan, but from debugging failures, questioning assumptions, and iterating relentlessly. The system is still experimental, but it already demonstrates that reinforcement learning can effectively control a noisy, real-world physical system like a slot car.
