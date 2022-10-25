# Base Loss Model
- The 156 significant fragment lengths used
- Tried out TSS specific bins only but gives sub-optimal performance
- Relative MAE loss with 0.01 constant
- Cannot claim 0.5% LoD
- 10 split correlation moderately good
- 0.5% LoD is pretty far away

# Learned Const Relative MAE
- The constant of the relative MAE loss is now part of model parameter
- Very important to initialize this constant weight with the right value (0.01 in our case) 
    - If you initialize with random value, then the constant can get stuck and your model will not be trained at all
- Very important not to set two different constants (to be learnt) in the numerator and denominator of the relative MAE loss
    - Because the model training will make the denominator constant extremely large to minimize the loss in that case 
- Approaching towards 0.5% LoD
- Hyperparameters to tune:
    - Relative MAE model constant initialization
    - Squared/ absolute value used in the custom loss constructed inside the model
    - Epoch no.

# Base Model Learned Const
- We saw in the previous learned const model that for most of the 10 splits, the final constant value after training is around 0.003
- We set our relative MAE constant value to 0.003
- Can easily claim LoD of 0.5% and also good performance at 0.1%
- Underestimation in tumour fraction prediction 
    - worse performance at the high end
    - can be solved using the selection model idea
- Risky -> training becomes unstable in rare cases and the model performance can be disastrous (around 0.09 MAE opposed to 0.04 MAE trend)

# Conditional Loss Based on True ctDNA Burden
- 3 loss functions used:
    - High loss: It is the L1 loss for samples with > 0.03 ctDNA burden
    - Mid loss: It is the relative MAE loss with 0.0085 constant value for samples with [0.01, 0.03] ctDNA burden
    - Low loss: It is the relative MAE loss with 0.003 constant value for samples with < 0.01 ctDNA burden
- I kept batch size of 1 and tried training each sample with appropriate loss function
    - Extremely slow training
    - MAE does not go down, nothing happens
- I created 3 training sets (low, mid and high ctDNA burden) and trained the model with them in cycles using appropriate loss
    - Fast training
    - MAE does not go down, nothing happens

# Learn_Const_Learn_Activation
- Here the idea of learning relative MAE constant has been kept in tact
- We added learned activation in each dense layer neuron:
    - x = F.relu(x) + self.begin_linear_rw * x * F.relu(x)
- The code above has some possible variants:
    - x = F.relu(x) + self.begin_linear_rw * x * F.relu(x)
    - x = F.relu(x) + self.begin_linear_rw * F.relu(x)
    - x = self.begin_linear_rw * x * F.relu(x)
    - x = self.begin_linear_rw * F.relu(x)
    - The first variant works the best and is provided in the STAN paper
- What if we add these learned activation functions in convolution layers as well?
    - Didn't work out well
- Why aren't we using tanh activation instead of relu just like the STAN paper?
    - Model is not able to learn anything
    - In a regression context, squishy functions such as sigmoid and tanh are not good at all
- VVI: NEVER USE RELU IN THE OUTPUT LAYER OF YOUR REGRESSION TASK, IT WILL MAKE YOUR TRAINING STUCK; ALWAYS USE LINEAR ACTIVATION

# Norm Tumour Learn Const Learn Activation
- The idea of learning relative MAE constant and learning activation function multiplicative co-efficient are kept in tact in this approach
- The ground truth tumour fractions of the training samples are z score normalized
- The model tries to predict these modified training sample outputs
- Training does not progress:
    - The idea of learning relative MAE constant and normalized tumour fraction prediction are contradictory
    - We choose not to normalize tumour fraction as it was tried before with improvement in the higher ctDNA burden sample prediction, but not in the lower ctDNA burden sample region

# Adaptive Sampling Learn Const Learn Activation
- The idea of learning relative MAE constant and learning activation function multiplicative co-efficient are kept in tact in this approach
- We integrate the idea of adaptive sampling of training samples from two papers:
    - Adaptive Self-supervision Algorithms for Physics-informed Neural Networks, arXiv, 2022
    - Efficient training of physics-informed neural networks via importance sampling, Computer-Aided Civil and Infrastructure Engineering, 2021
- Variants to try out:
    - No uniform sampling, only non-uniform sampling:
        - try without any cosine annealing -> pure probability distribution based
        - try cosine annealing with different time periods
        - try out different epoch numbers
    - uniform sampling and then non-uniform sampling cycles:
        - try out different epochs for both scenarios
    - cycling uniform sampling and non-uniform sampling cycle
    - Different partition widths

# Final configuration:
- learned activation and learned constant of relative MAE kept
- Uniform sampling of training samples in the first 70 epochs
- Non-uniform sampling according to training sample partition loss based probability for the next 30 epochs. In these 30 epochs:
    - Cycling between uniform and non-uniform sampling done using cosine annealing where cycle length is 5 epochs
    - Training sample partitioning done according to true tumour fraction (15 partitions in total of almost equal size (around 115))
    - The relative MAE constant kept frozen during these 30 epochs of training (else the adaptive sampling and constant learning battle each other giving sub-optimal performance)

# Conclusion:
- Need 5X WGS data to get over noise -> hopeful of getting 0.1% LoD.