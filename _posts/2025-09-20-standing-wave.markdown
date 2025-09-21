---
layout: post
title: "Modeling Microwave Heating"
subtitle: Visualizing standing waves and microwave heating with simple Python models
date:   2025-09-20 12:00:00 -0700
categories: misc
tags: [microwave, physics, simulation, visualization, python, pytorch, matplotlib, plotly]
author: Yurii Chukhrai
---

## Introduction

It started with a casual discussion with a friend: how exactly does a microwave oven heat food? This everyday question led me down a path of curiosity—how large language models (LLMs) and some coding experiments can help build simple, visual models of what happens inside a microwave.
During this exploration, I came across an excellent explanation by Bill Hammack (the **Engineer Guy**) in his video [How a Microwave Oven Works](https://www.youtube.com/watch?v=kp33ZprO0Ck). His clear physical intuition inspired me to experiment further and try build graphical representation of simple mathematical models.
The result: three different simulations, each one capturing a different perspective of standing waves inside a microwave cavity.
None of these are fully realistic physics solvers, but they serve as educational demonstrations of how electromagnetic waves might look in simplified scenarios.

## Three Models of Microwave Heating
I experimented with three progressively more visual models. Each uses Python and a mix of visualization libraries to illustrate wave patterns.

### 2D Static Model of a Standing Wave (PyTorch + Matplotlib)

This model generates a two-dimensional standing wave pattern using sine functions along both the x and y directions. It produces a grid where the amplitude oscillates, resembling the stationary interference of waves inside a cavity.

#### Key Idea
Physically, it represents the **electric field distribution** of a standing wave at a single instant in time. The wave number `k` is set to 1 for simplicity, meaning the pattern is illustrative rather than tuned to a real cavity.

#### Limitations
 * No time evolution (just a snapshot).
 * No physical units or normalization.
 * Doesn’t model the actual geometry of a microwave oven.

![Example#1](/resources/misc/2025-09-20-standing-wave/example_01.jpg){:.mx-auto.d-block :}

<details markdown="1">
<summary>Code</summary>

Routine importing libraries.
```python
#hide

import os
os.environ['PYTORCH_ENABLE_MPS_FALLBACK'] = '1'

from fastai.torch_core import defaults
import torch

# Set the default device to MPS for fastai
if torch.backends.mps.is_available():
    defaults.device = torch.device('mps')
    print("Using MPS device.")
else:
    print("MPS not available. Falling back to CPU.")
```

```python
# 1. Example code (PyTorch + Matplotlib, simple standing wave)
# This code models a 2D standing wave pattern using sine functions along the x and y directions, producing a grid of oscillating amplitudes. 
# Physically, it represents the spatial distribution of a wave’s electric field (or another scalar field) at a fixed moment in time. 
# The wave number k is simplified to 1, so the pattern is illustrative rather than matching a real material or boundary condition. 
# From a physics standpoint, the math is correct for a standing wave snapshot, but it omits time dependence, normalization, and physical units, making it a conceptual demonstration rather than a fully realistic simulation.

import matplotlib.pyplot as plt
from mpl_toolkits.mplot3d import Axes3D

# Grid
N = 100
x = torch.linspace(0, 2*torch.pi, N)
y = torch.linspace(0, 2*torch.pi, N)
X, Y = torch.meshgrid(x, y, indexing="ij")

# Parameters
k = 1.0   # wave number (simplified)
t = 0.0   # static snapshot

# Standing wave pattern (2D slice)
E = torch.sin(k*X) * torch.sin(k*Y)

# Plot
fig = plt.figure(figsize=(8,6))
ax = fig.add_subplot(111, projection='3d')
ax.plot_surface(X.numpy(), Y.numpy(), E.numpy(), cmap='viridis')

# Add this line to set a proportional aspect ratio
ax.set_box_aspect([1, 1, 1])

ax.set_xlabel('x')
ax.set_ylabel('y')
ax.set_zlabel('E-field amplitude')

# Automatically adjust subplot params for a tight layout
plt.tight_layout()

plt.show()
```

</details>


### 3D Animated Microwave Cavity (PyTorch + Matplotlib + HTML5)
Here, the model simulates a 3D standing wave field inside a cubic cavity. 
The electric field is represented as a product of sinusoidal functions in `x`, `y`, and `z`, with oscillation in time governed by angular frequency `ω`.

#### Key Idea
This resembles the spatial mode structure inside a resonant cavity, where nodes (no oscillation) and antinodes (maximum oscillation) form due to interference.

#### Limitations
 * Oversimplified boundary conditions.
 * No material properties or real mode indices.
 * Serves more as a visual sketch than a quantitative model.

[![Example#2](https://img.youtube.com/vi/8p1vDvgB0V4/0.jpg)](https://www.youtube.com/shorts/8p1vDvgB0V4 "Example#2")

<details markdown="1">
<summary>Code</summary>
This produces a 3D oscillating field, which can be animated in Jupyter or exported as HTML5 for interactive viewing.

```python
# 2. Example Code (3D Animated Microwave Cavity with PyTorch)
# This code simulates a 3D standing wave field inside a cubic microwave cavity, where the electric field is modeled as a product of sinusoidal functions in 
# x, y, z and oscillates in time with frequency ω. 
# Physically, it captures the qualitative shape of cavity modes (nodes and antinodes), but uses a simplified form without boundary conditions, mode indices, or material effects. 
# It is more of a conceptual visualization than a quantitatively accurate cavity simulation.
    
import torch
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.animation import FuncAnimation
from IPython.display import HTML

# Set the animation size limit to a higher value (e.g., 500 MB)
plt.rcParams['animation.embed_limit'] = 500

# Grid
N = 30
x = torch.linspace(0, 2*torch.pi, N)
y = torch.linspace(0, 2*torch.pi, N)
z = torch.linspace(0, 2*torch.pi, N)
X, Y, Z = torch.meshgrid(x, y, z, indexing="ij")

# Parameters
k = 1.0     # wave number (scaled)
omega = 2.0 # angular frequency

# Figure
fig = plt.figure(figsize=(7,6))
ax = fig.add_subplot(111, projection="3d")

# Initial field
t0 = 0
E = torch.sin(k*X) * torch.sin(k*Y) * torch.sin(k*Z) * torch.cos(torch.tensor(omega*t0))

# Scatter plot (sample points where intensity is strong)
scatter = ax.scatter(X.numpy().flatten(), 
                     Y.numpy().flatten(), 
                     Z.numpy().flatten(), 
                     c=E.numpy().flatten(), 
                     cmap="plasma", alpha=0.4)

ax.set_xlabel("x")
ax.set_ylabel("y")
ax.set_zlabel("z")
ax.set_title("Microwave. Standing Waves")

# Update function for animation
def update(frame):
    t = frame * 0.07
    E = torch.sin(k*X) * torch.sin(k*Y) * torch.sin(k*Z) * torch.cos(torch.tensor(omega*t))
    scatter.set_array(E.numpy().flatten())
    return scatter,
 
# Animate
ani = FuncAnimation(fig, update, frames=100, interval=200, blit=False)

#plt.show()

HTML(ani.to_jshtml())
```

</details>

### 2D Animated Surface of a Standing Wave (Plotly)

The last model creates an animated 3D surface that evolves over time, showing how the standing wave oscillates on a two-dimensional plane.

#### Key Idea
This is similar to watching a vibrating membrane or drumhead. The animation makes nodes and antinodes visible as areas of stillness and large motion.

#### Limitations
 * No damping, boundaries, or material effects.
 * Parameters are chosen for clarity, not realism.

[![Example#3](https://img.youtube.com/vi/GE4tNo1qZAM/0.jpg)](https://youtu.be/GE4tNo1qZAM "Example#3")

<details markdown="1">
<summary>Code</summary>

```python
# 3. Example. 
# This code creates an animated 3D surface of a 2D standing wave, where the displacement depends on sinusoidal functions in both 
# x and y directions and oscillates in time with frequency ω.
# Physically, it represents the vibration of a membrane or wave on a bounded surface, showing nodes and antinodes characteristic of standing waves. 
# It uses simplified parameters (no damping, no real boundary conditions, no units), so it serves as a demonstrative model rather than an exact physical simulation.

import numpy as np
import plotly.graph_objects as go

# 1. Grid parameters
N = 50
x = np.linspace(0, 2 * np.pi, N)
y = np.linspace(0, 2 * np.pi, N)
X, Y = np.meshgrid(x, y)

# 2. Standing wave parameters
kx = 1.0
ky = 1.5
omega = 2.0

def standing_wave_surface(t):
    return np.sin(kx * X) * np.sin(ky * Y) * np.cos(omega * t)

# 3. Number of frames for animation
num_frames = 100
times = np.linspace(0, 2 * np.pi / omega, num_frames)

# Fixed scene (lock axis ranges!)
fixed_scene = dict(
    xaxis=dict(title='X', range=[0, 2*np.pi]),
    yaxis=dict(title='Y', range=[0, 2*np.pi]),
    zaxis=dict(title='Amplitude', range=[-1.1, 1.1]),
    aspectmode='cube',   # keep scaling consistent
    camera=dict(
        up=dict(x=0, y=0, z=1),
        center=dict(x=0, y=0, z=0),
        eye=dict(x=1.25, y=-1.25, z=1.25)
    )
)

# 4. Frames
frames = []
for i, t in enumerate(times):
    Z_data = standing_wave_surface(t)
    frames.append(go.Frame(
        data=[go.Surface(z=Z_data, x=X, y=Y, colorscale='Viridis', cmin=-1, cmax=1)],
        name=f'frame_{i}',
        layout=go.Layout(scene=fixed_scene)   # 👈 lock scene for every frame
    ))

# 5. Initial figure
initial_Z = standing_wave_surface(times[0])
fig = go.Figure(
    data=[go.Surface(z=initial_Z, x=X, y=Y, colorscale='Viridis', cmin=-1, cmax=1)],
    layout=go.Layout(
        title=dict(
            text='Surface motion of a standing wave',
            x=0.5,  # center title
            font=dict(size=22)
        ),
        scene=fixed_scene,
        width=900,   # 👈 wider
        height=700,  # 👈 taller
        margin=dict(l=20, r=20, t=80, b=50),  # 👈 less padding = more space for plot
        font=dict(size=14),  # larger axis/tick labels
        updatemenus=[dict(
            type="buttons",
            buttons=[dict(label="▶️ Play",
                          method="animate",
                          args=[None, {"frame": {"duration": 50, "redraw": True},
                                       "fromcurrent": True, "transition": {"duration": 0}}]),
                     dict(label="⏸️ Pause",
                          method="animate",
                          args=[[None], {"frame": {"duration": 0, "redraw": True},
                                         "mode": "immediate"}])
                     ],
            direction="left",
            x=0.1, xanchor="right", y=0, yanchor="top"
        )],
        sliders=[dict(
            steps=[dict(method='animate',
                        args=[[f'frame_{i}'],
                              {"frame": {"duration": 0, "redraw": True},
                               "mode": "immediate"}],
                        label=f'{i}')
                   for i in range(num_frames)],
            x=0.1, len=0.9,
            currentvalue={'visible': True, 'prefix': 'Frame: '}
        )]
    ),
    frames=frames
)

fig.show()
```

</details>

---

## Summary

These experiments are not precise engineering simulations—they lack material absorption, cavity geometry, and realistic wave numbers.
Yet, they illustrate something valuable:
 * Complex physical processes can be explored with relatively simple models.
 * Visualization bridges intuition and equations. Even simplified models can help us “see” the invisible dynamics of electromagnetic waves.
 * LLMs and coding experiments accelerate exploration. They don’t replace deep physics, but they make prototyping models more accessible.

So, a conversation about heating leftovers turned into an exploration of physics, code, and visualization. 
By modeling simplified standing waves, we gain an intuitive glimpse into what might be happening each time we push that microwave start button.