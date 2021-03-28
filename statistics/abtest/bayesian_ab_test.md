# Bayesian A/B Testing

다음 아티클을 읽고 간단히 정리해보자.

[Chris Stucchio | Bayesian A/B Testing at VWO](https://cdn2.hubspot.net/hubfs/310840/VWO_SmartStats_technical_whitepaper.pdf)

# Problem

A/B 테스트를 할 때마다 사람들이 가장 궁금해하는 것 중에 하나는 B안이 A안보다 좋을 확률이 얼마나 되냐는 것이다. 일반적으로 사용하는 frequentist 방법론으로는 이러한 질문에 대한 답을 할 수 없다. frequentist 방법론들은 보통 이런 방법을 따른다.

1. 귀무가설(Null Hypothesis)을 선택한다. 보통 A, B안 사이에 차이가 없을 것이라고 가정하게 된다.
2. 이제 실험을 수행하고, 통계량을 구한다.
3. p-value를 계산한다. p-value는 귀무가설이 맞다고 가정했을 때, 현재보다 더 극단적인 케이스의 통계량 수치가 나올 확률을 의미한다.
    - p-value가 의미하는 것은 다음과 같다.
    - "동일한 샘플 크기로 A/A 테스트를 수행할 경우, 방금 본 결과와 같거나 더 극단적인 결과가 나올 확률이 p값보다 작다."

p-value 는 B안이 A안보다 좋을 확률을 의미하는 것이 아니지만, 그렇게 사용되는 경우가 종종 있다.

흔히 사용하는 A/B 테스트 방식은 false positive 를 발생시키기 쉽다. 예를 들어, 여러 가지 목표를 만들어 두고 테스트가 끝날 때 특정 목표를 선택해버릴 수도 있다. 

# The Joint Posterior for 2 variants

A안과 B안이라는 2개의 선택지가 있다고 해보자. 이 때, joint posterior는 다음과 같다.

```
P(lambda_A, lambda_B) = P_A(lambda_A) * P_B(lambda_B)

lambda_A : A안의 true conversion rate
lambda_B : B안의 true conversion rate
P_A(lambda_A) : A안에 대한 Posterior distribution
P_B(lambda_B) : B안에 대한 Posterior distribution
```

Joint posterior 를 알고있으면 다양한 값을 구할 수 있다. 그 중에서 가장 중요한 것은 loss function 이다.
이 함수를 바탕으로 어떤 안을 선택할지, 언제 테스트를 멈출지 결정할 것이다.

## Chance to beat control

데이터를 통해 증거를 모으고 그 결과 A안을 선택했다고 해보자. 우리가 실수했을 가능성은 얼마나 될까? 
Error Probability는 다음과 같이 정의할 수 있다.

```
E[I](A) = \int_{0}^{1} \int_{0}^{\lambda_A} P(\lambda_A, \lambda_B) d\lambda_B \lambda_A
```

이 정의는 테스트를 지속해야 하는지를 결정할 수 있는 꽤 직관적인 선택으로 보인다. 하지만 이 지표는 결정을 내리는데 있어 중요한 결함이 있다. 
바로 **모든 에러를 동일하게 취급한다는** 점이다.

## The Loss Function

Loss Function 은 중요한 부분에서 발생한 에러일수록 더 큰 영향을 미치도록 에러 함수를 수정한다. 
여기서는 Loss Function 을 `lambda_A` , `lambda_B` 값이 주어졌을 때 A 또는 B안을 선택하여 발생할 것으로 예상되는 손실의 양을 나타나는 함수로 정의한다.

```
L(lambda_A, lambda_B, A) = max(lambda_B - lambda_A, 0)
L(lambda_A, lambda_B, B) = max(lambda_A - lambda_B, 0)
```

- Example : 다음과 같은 상황을 가정해보자.
    - A안의 전환율은 `lambda_A = 0.1` , B안의 전환율은 `lambda_B = 0.15` 로 알려져있다.
    - A안을 선택할 경우 loss는 `max(0.15 - 0.1, 0) = 0.05` 이고,
    - B안을 선택할 경우 loss는 `max(0.1 - 0.15, 0) = 0.0` 이다.

이제 **joint posterior 가 주어졌을 때의 기대 손실은 loss function의 기대값이다.**

```
E[L](?) = \int_{0}^{1} \int_{0}^{\lambda_A} L(\lambda_A, \lambda_B, ?) P(\lambda_A, \lambda_B) d\lambda_B \lambda_A
```

## Computing the loss and error functions

error, loss function을 수치적으로 구하는 방법은 직관적이다. 

```python
# (1) Computing the joint posterior
# 다음과 같이 2개의 posterior를 가지고 있다고 생각해보자
posterior_A = []
posterior_B = []

# joint posteior는 2d array로 나타낸다
joint_posterior = zeros(shape=(100,100))

for i in range(100):
    for j in range(100):
        joint_posterior[i, j] = posterior_A[i] * posterior_B[j]

# (2) Computing the error function
error_function_A = 0
for i in range(100):
    for j in range(i, 100):
        error_function_A += joint_posterior[i, j]

error_function_B = 0
for i in range(100):
    for j in range(0, i):
        error_function_B += joint_posterior[i, j]

# (3) Computing the loss function
def loss(i, j, var):
    if var == 'A':
        return max(j*0.01 - i*0.01, 0)
    if var == 'B':
        return max(i*0.01 - j*0.01, 0)

loss_function = 0
for i in range(100):
    for j in range(100):
        loss_function += joint_posterior[i, j] * loss(i, j, 'A')
```

# Running a Bayesian A/B test

3개 이상의 케이스를 다루기 전에, A/B 테스트가 어떻게 동작할지 살펴보자. 기본적인 아이디어는 다음과 같다.

1. 실험에서 목표로 하는 에러 허용치 `e` 를 정한다. 
2. 기대 loss 값이 사전에 설정해둔 에러 허용치보다 낮아질 때까지 실험을 계속한다.

`e` 는 보통 백분율 형태로 해석한다. 우리가 잘못된 선택을 했을 때, 해당 선택을 함으로써 발생하는 손실을 나타낸다. 따라서 발생하더라도 크게 신경쓰이지 않을 정도로 낮은 숫자를 설정해야 한다.

- Example : 두 개의 버튼에 다른 색상을 적용하는 테스트를 가정해보자.
    - 10% 정도 상승할 수 있을 것이라고 기대하고 있다.
    - 반대로 지표가 나빠진다면 0.2% 이내로 하락할 수 있다. 이 정도 하락하는 것은 크게 문제가 되지 않는다.
    - 따라서 이 경우에는 `e = 0.002` 로 설정할 수 있다.

**베이지안 A/B 테스트의 구체적인 프로세스는 다음과 같다.**

1. A안과 B안 중 하나를 랜덤하게 유저에게 보여주는 실험을 시행한다.
2. 주기적으로 통계 지표 `n_A` , `c_A` , `n_B` , `c_B` 를 수집한다.
    - `n_A` : A안이 노출된 횟수
    - `n_B` : B안이 노출된 횟수
    - `c_A` : A안이 전환된 횟수
    - `c_B` : B안이 전환된 횟수
3. `E[L](A)` 와 `E[L](B)` (A, B안의 기대 손실)을 계산한다.
4. A안과 B안의 기대 손실 중 에러 허용치 `e` 보다 작은 값이 있는지 확인한다.
    - 없는 경우 → 1번으로 돌아간다.
    - 있는 경우 → **실험을 멈추고 기대 손실이 `e` 보다 작은 안을 선택한다.**
