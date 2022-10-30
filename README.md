# zkapp

zkapp with snarkjs like iden3

### 1. 설치!

npm 으로 snarkjs 모듈을 설치한다.

`npm install -g snarkjs@latest`

그리고 snarkjs 의 `groth16 prove`, `g16p` --help 옵션을 확인한 후

a power of tau ceremony 를 이용하는 명령어를 입력한다.
사실 나는 그게 뭔지 잘 모르겠다. `bn128` 과 `bls12-381` 를 지원한다는데...!?
숫자 12는 constraints 수를 2의 최대 12승까지로 설정한다는 것이다. 지원하는 최대 설정값은 28이다.

`snarkjs powersoftau new bn128 12 pot12_0000.ptau -v`

### 2. 엔트로피 설정

`snarkjs powersoftau contribute pot12_0000.ptau pot12_0001.ptau --name="First contribution" -v`

위 명령어를 입력하면 내가 설정할 수 있는 랜덤 시드 같은 문구를 입력하는 창이 나오는데
나는 그냥 내 정체성 키워드를 입력했다! 뭔지는 시크릿임.
