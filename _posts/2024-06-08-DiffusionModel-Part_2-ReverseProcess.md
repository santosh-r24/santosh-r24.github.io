---
layout: post
title: Reverse Pass, Part 2 - Design fundmanentals of Diffusion Models
categories: [Gen AI, Diffusion Model]
---

## Introduction

Continuing from where we left off in Part 1, we’re going to talk about the next integral process where training happens: The Reverse Process. The focus is on training the model component (denoising network) to remove noise and finally arrive at \(X_0\) (the original image). The design components governing this process are:

### 1. Network Architecture

The two popular architectures used are U-Net and Transformer because of their capacity to model complex relationships in a wide range of applications:

**U-Net**  
This is a U-shaped encoder-decoder architecture. It creates a bottleneck of information, which forces the network to learn features effectively.
- **Encoder**: It extracts high-level data and sends it to a downsampling layer for compression.
- **Decoder**: Uses the above features based on specific applications.

**Transformer**  
This architecture also has an encoder/decoder structure.
- **Encoder**: Consists of a self-attention layer, which measures relationship between data, regardless of spatial location. Hence, it’s better able to capture global dependencies (e.g., the overall shape of an object). However, being able to capture these global relationships comes at the cost of losing short-range relationships(the minute details of specific objects). To combat this, some U-net structures are added to maintain this aspect as well.
- **Decoder**: Uses the above features on specific applications. 
### 2. Parameterization of Reverse Mean

Learning the patterns of how the model adds and removes noise, i.e., the mean, will help with faster convergence and hence quicker reverse process, leading to faster training times. Calculating the Reverse Mean (μ) helps the model determine the expected value at each timestep to accurately denoise the image.

\[ \mu(xt, t) := \frac{\sqrt{\alpha_{t-1}(1 - \alpha_{t-1})} \cdot xt + \sqrt{\alpha_{t-1}(1 - \alpha_t)} \cdot x_0}{1 - \alpha_{t-1}} \]

- $\alpha$ - Variance Schedule parameters

- $x_{t}$ - Noisy data at time step t. 

- $x_{o}$ - Original data


There are two ways of parameterization:
- **Direct Parameterization**: Calculate μ by replacing \(x_0\) (actual \(X_0\)) with \(\hat{x_0}\) (the model’s estimate of \(X_0\) during the reverse process).
- **Indirect Parameterization**: 
  - **Residual Noise**: This predicts the noise added in the forward process and subtracts this from the image to arrive at \(X_0\). This estimates the noise added. Since there’s a fixed range of noise values, it leads to a consistent prediction of magnitude.

      \[ \mu_\theta(xt, t) := \frac{1}{\sqrt{\alpha_t}} \cdot xt - \frac{(1 - \alpha_t)}{\sqrt{\alpha_t(1 - \alpha_{t-1})}} \cdot \hat{\epsilon}_t \]
      
      \(\hat{\epsilon}_t\) - Predicted Noise
    
      \(\epsilon_t\) - Actual Noise added in the Forward process at that time step

  - **Score**: This is the gradient of the log of the distribution of noise. It indicates the most possible change between two timesteps by guiding the denoising trajectory with direction and magnitude. This is better for later stage refinements to denoise accuracy.

      \[ dx = f(x, t) - g^2(t) \cdot \hat{s}_t \cdot dt + g(t) \cdot dw \]

      ![Score]({{ site.url }}/images/Score_visualization.png "Graph of Score")
      
In practice, a combination of different parameterizations is used. At the start of the reverse process, when there’s a lot of noise, direct parameterization is used. This helps the network to understand the global structure. At a later stage, since more refining is needed, an indirect method like noise or score is used for accurate denoising.

### 3. Optimization Design on Learning Objective

The learning objective is the function that the model aims to optimize during training. There are different strategies involved to improve the effectiveness of the denoising network.

- **Reverse Variance**  
  An appropriate reverse variance minimizes the discrepancy between the forward and reverse processes, reducing the time steps required to converge. There are three optimization techniques for finding an appropriate reverse variance value:
  - Using empirical values for each timestep.
  - A predefined schedule that dictates how variance changes at each timestep.
  - Estimating optimal variance by analytically arriving at it from the predicted score.

- **Learning Weight**  
  This is used to balance learning priorities in training, controlling the focus on global vs. local details of data at different stages of the reverse process. There are two optimization techniques for finding an appropriate learning weight value:
  - Using predefined noise schedules directly as learning weight, which emphasizes learning global structures early on.
  - Computing weights based on Signal to Noise Ratio (SNR). This takes into account the remaining noise present at each timestep. This is a more accurate way of denoising because it focuses a lot on the local data.

### Mindmap 
![RP Mind Map]({{ site.url }}/images/Reverse_Snap.png "Mind Map")
