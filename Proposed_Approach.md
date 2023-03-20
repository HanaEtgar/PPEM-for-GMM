# Proposed Approach  
  
## Motivation  
All proposed approaches in prior work on PPEM involve a trade-off between accuracy, privacy, and performance.  
For example, Fully Homomorphic Encryption (FHE) based algorithms can be time consuming and computationally heavy as computing over encrypted data
incurs a high computational overhead, however privacy will be maintained as FHE allows for secure computations on encrypted data 
without requiring access to the plaintext.  
Another example is Secure Multi-Party Computation (SMPC) based PPEM: maintaining privacy is a primary goal of SMPC. However, achieving perfect privacy comes
at the cost of computational complexity and performance, and the efficiency of SMPC can be impacted by the number of parties involved in the computation.  
  
  
## Our Contribution  
We propose a protocol for privacy-preserving expectation maximization that attains the desirable properties of FHE 
while simplifying calculations to only require addition operations on encrypted data. 
This approach allows us to achieve a high level of privacy while also improving performance and reducing computational overhead.  
By utilizing the strengths of FHE while streamlining the computation process, 
we are able to strike a balance between accuracy, privacy, and performance in our proposed approach.  
  
  
## Setup
We address the following scenario:  
- Data is distributed and private: there are $n$ parties, each holding its own private data $x_i$ (a two-dimensional point) and 
we wish to fit a GMM to the full dataset, without revealing the private data of each party.  
- We propose a client-server model, where the cloud service is an untrusted third party and acts as the central server, providing a service to the clients who are the owners of the private data.  
- We assume an Honest-but-Curious adversary and do not consider the Malicious case, thus we can assume that the server (the untrusted third party, the "cloud") is not mailcious.  
- To generate, distribute, and manage cryptographic keys for the CKKS scheme via TenSEAL library, we utilize a Key Management Service (KMS).  
  
  
## Reminder: Distributed Expectation-Maximization for Gaussian Mixture Models  
Consider the scenario where $n$ parties each has its own data $x_i$ and these parties would like to collaborate to learn a GMM based on the full dataset { $x_1, x_2, ..., x_n$ }.  
Assuming there are $c$ Gaussian components, the GMM density is given by:  
$$p(x)=\Sigma_{j=1}^c \beta_j p(x| \mu_j, \Sigma_j)$$ 
where $\beta_j$ is the mixing coefficient of the $j^{th}$ Gaussian component, and $\mu_j$ and $\Sigma_j$ are the mean and covariance, respectively, of the $j^{th}$ Gaussian component.  
  
**The EM algorithm is iterative and for each iteration $t$, the following steps are taken for all Gaussian components:**  
$\boldsymbol{E-Step:}$  
$$P(x_i|N_j^t) = \frac{p(x_i|\mu_j, \Sigma_j) \beta_j^t}{ \Sigma_{k=1}^c p(x_i|\mu_k, \Sigma_k)\beta_k^t}$$  
where $p(x_i|\mu_k, \Sigma_k)$ is the pdf for a Gaussian distribution with mean $\mu_k$ and covariance matrix $\Sigma_k$, and $\beta_k$ is the mixing coefficient of the $k^{th}$ Gaussian component.  
  
$\boldsymbol{M-Step:}$    

$$\beta_j^{t+1} = \frac{\Sigma_{i=1}^n P(x_i|N_j^t)}{n}$$   
  
$$\mu_j^{t+1} = \frac{\Sigma_{i=1}^n P(x_i|N_j^t) x_i}{\Sigma_{i=1}^n P(x_i|N_j^t)}$$  
  
$$\Sigma_j^{t+1} = \frac{\Sigma_{i=1}^n P(x_i|N_j^t) (x_i-μ_j^t) (x_i-μ_j^t)^\top}{\Sigma_{i=1}^n P(x_i|N_j^t)}$$  
  
where $P(x_i|N_j^t)$ denotes the conditional probability that data $x_i$ belongs to Gaussian model $j$ (result of the E-Step).  


## Our Approach  
It's worth noting that in the distributed version of the Expectation Maximization algorithm, the $\{E-Step}$ can be computed locally. 
This means that each party can perform the E-step on its own data, thus preserving data privacy.  
However, the $\{M-Step}$ requires data aggregation to compute global updates of the Gaussian components' parameters, which raises privacy concerns.  
To address this, we simplify the M-step by breaking it down into intermediate updates that are also computed locally and will periodically be communicated with the central server.  
  
### The Algorithm  
For each iteration $t$, the KMS generates a pair of public and secret keys $(pk_t, sk_t)$ and distributes both keys to all parties involved in the computation. Therefore, each party will hold both the public key $pk_t$ and the corresponding secret key $sk_t$ for the current iteration. Furthermore, the server gets access only to the public key $pk_t$.  
  
For every Gaussian component $j$, each node $i$ computes the following intermediate updates:  
$$a_{ij}^t = P(x_i|N_j^t)$$ 
$$b_{ij}^t = P(x_i|N_j^t)x_i$$
$$c_{ij}^t = P(x_i|N_j^t)(x_i - \mu_j^t)(x_i - \mu_j^t)^\top$$  
  
All the above updates can also be computed locally at node $i$, and then the node can compute its intermediate updates vector:  
$$v_{ij}^t=[a_{ij}^t,  b_{ij_{0}}^t, b_{ij_{1}}^t, c_{ij_{00}}^t, c_{ij_{01}}^t, c_{ij_{10}}^t, c_{ij_{11}}^t]$$  

$x_i$ is a two-dimensional point, $a_{ij}$ is a scalar, $b_{ij}$ is a two-dimensional vector, and $c_{ij}$ is a 2x2 matrix. Thus, $v_{ij}^t$ includes all the entries of $a$, $b$, and $c$.  
  
Node $i$ then encrypts this vector and sends the ciphertext $\hat{v_{ij}^t} ← Enc_{pk_t}(v_{ij}^t)$ to the server.  
After receiving these intermediate updates from all nodes, the server computes the sum of all the vectors (for a specific Gaussian component $j$):  
$$\hat{v_{j}^t} ← Eval_{pk_t}(C, \hat{v_{i_{1}j}^t}, ..., \hat{v_{i_{n}j}^t})$$
where $C$ is a homomorphic addition circuit.  
  
The server sends back the result $\hat{s_{ij}^t}$ to all the nodes, and then every node $i$ decrypts it to obtain the sum
${v_{j}^t} ← Dec_{sk_t}(\hat{v_{j}^t)}$ which is $v_j^t=\Sigma_{i=1}^n v_{ij}^t$, 
from which we can obtain the sums $\Sigma_{i=1}^n a_{ij}^t,  \Sigma_{i=1}^n b_{ij}^t,  \Sigma_{i=1}^n c_{ij}^t$.  
  
These sums are used to update the global estimates of the mixture weights and the means and covariances of each Gaussian component (global updates):  

$$\beta_j^{t+1} = \frac{\Sigma_{i=1}^n P(x_i|N_j^t)}{n} = \frac{\Sigma_{i=1}^n a_{ij}^t}{n}$$   
  
$$\mu_j^{t+1} = \frac{\Sigma_{i=1}^n P(x_i|N_j^t) x_i}{\Sigma_{i=1}^n P(x_i|N_j^t)} = \frac{\Sigma_{i=1}^n b_{ij}^t}{\Sigma_{i=1}^n a_{ij}^t}$$  
  
$$\Sigma_j^{t+1} = \frac{\Sigma_{i=1}^n P(x_i|N_j^t) (x_i-μ_j^t) (x_i-μ_j^t)^\top}{\Sigma_{i=1}^n P(x_i|N_j^t)} = \frac{\Sigma_{i=1}^n c_{ij}^t}{\Sigma_{i=1}^n a_{ij}^t}$$  
  
  
  
## Privacy  
Let us quantify the individual privacy of each node’s private data using mutual information $I(\overline{X};\overline{Y})$ which measures the mutual dependence between two random variables $\overline{X}, \overline{Y}$. We have that $0 \leq I(\overline{X};\overline{Y})$, with equality if and only if $\overline{X}$ is independent of $\overline{Y}$ . On the other hand, if $\overline{Y}$ uniquely determines $\overline{X}$ we have $I(\overline{X};\overline{Y})=I(\overline{X};\overline{X})$ which is maximal.  
  
Suppose each node shared its intermediate updates vector without encrypting it first, then although the node does not share the private date directly, it is still revealed to the server. This is because with the intermediate updates $a_{ij}^t$ and $b_{ij}^t$, the server is able to determine the private data $x_i$ of each node
$i$ since $b_{ij}^t = a_{ij}^t x_i$.  
That is, at each iteration the server has the following mutual information
$$I(\overline{X_i};\overline{A_{ij}^t}, \overline{B_{ij}^t}) = I(\overline{X_i};\overline{X_i})$$
which is maximal. This means that every node’s private data $x_i$ is completely revealed to the server. Hence, the algorithm would not be privacy-preserving at all.  
  
  
In our approach, the server is unable to determine the private data $x_i$ of each node as it only receives the encrypted intermediate updates $a_{ij}^t$ and $b_{ij}^t$.  
Moreover, at the end of each iteration, each node solely acquires the global updates, which do not expose any information regarding the private data of other nodes. Specifically, from the global updates, nodes can acquire $\Sigma_{i=1}^n a_{ij}^t$, $\Sigma_{i=1}^n b_{ij}^t$, and $\Sigma_{i=1}^n c_{ij}^t$, from which they can compute the sum of the intermediate updates of other nodes. Given the large number of participants, this does not raise any privacy concerns.


## Correctness  
The fitted model, i.e., the estimated parameters of GMM, should be the same as using non-privacy preserving counterparts. Namely, the performance of the
GMM should not be compromised by considering privacy.  
Since we're using the CKKS scheme, the results obtained through this scheme may be approximate, but this does not significantly compromise the accuracy of the model. 
  

## Results  
In this section, we demonstrate numerical results to validate the comparison between the non-privacy preserving algorithm and our privacy preserving approach.  
  
To achieve this, we utilize a data generation function that takes three input parameters: `num_of_gaussians`, which determines the number of Gaussian components to simulate, `points_per_gaussian`, which determines the number of data points in each Gaussian component, and `mean_range`, which specifies the range of mean values for the data points.   
  
We simulate several GMMs using the data generation function, and compare the two algorithms using the log-likelihood of the GMMs. We plot the log-likelihood as a function of the iteration number of our proposed privacy preserving approach and the existing non-privacy preserving approach.  
  
The log-likelihood of a GMM with $n$ data points and $c$ Gaussian components is defined by   
<img width="257" alt="image" src="https://user-images.githubusercontent.com/100927079/226475142-f3910689-f92b-4fc1-b018-f5c0fcf3fba5.png">  

  
- **Experiment 1: `num_of_gaussians=3, points_per_gaussian=1000, mean_range=[-20, 20]`**  
![image](https://user-images.githubusercontent.com/100927079/226111471-01d37823-fc67-465b-bc24-5009ef703f76.png)  
  
- **Experiment 2: `num_of_gaussians=5, points_per_gaussian=900, mean_range=[-10, 10]`**  
![image](https://user-images.githubusercontent.com/100927079/226111547-23f18ae8-5872-4831-aa43-c1c5ec8cfcf4.png)  
  
- **Experiment 3: `num_of_gaussians=4, points_per_gaussian=500, mean_range= [-10, 10]`**  
![image](https://user-images.githubusercontent.com/100927079/226111572-8838927e-7eec-499b-ab91-86432dcf440b.png)  
  
-  **Experiment 4: `num_of_gaussians=5, points_per_gaussian=500, mean_range=[-10, 10]`**  
![image](https://user-images.githubusercontent.com/100927079/226111605-66616593-aac1-4a7e-909e-de9684857b67.png)  
  
- **Experiment 5: `num_of_gaussians=2, points_per_gaussian=1000, mean_range=[-10, 10]`**  
![image](https://user-images.githubusercontent.com/100927079/226111657-8b122437-d86a-4883-bfdd-639c8b2a70f7.png)  
  
- **Experiment 6: `num_of_gaussians=3, points_per_gaussian=1000, mean_range=[-10, 10]`**  
![image](https://user-images.githubusercontent.com/100927079/226111719-04a5760d-6a4d-46be-8fe4-6f3837c6210d.png)   
  
  
  
## Conclusion and Further Questions  
As shown in the figures in the previous seciton, we see that the proposed approach yields parameter estimations for GMMs that are indistinguishable from those of the non-privacy preserving approach. Hence, the output correctness of the proposed approach is guaranteed and not compromised by considering privacy.  
By using fully homomorphic encryption, we were able to maintain participants' individual privacy because the private data did not need to be disclosed for computations.  
Regarding performance, applying parallel computing could significantly enhance it. Instead of having the server wait for all participants to send intermediate updates, the process can be done simultaneously, resulting in a faster runtime.  







  
  


  

