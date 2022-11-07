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
=> pot12_0000.ptau 생성

### 2. 엔트로피 설정

`snarkjs powersoftau contribute pot12_0000.ptau pot12_0001.ptau --name="First contribution" -v`
=> pot12_0001.ptau 생성

위 명령어를 입력하면 내가 설정할 수 있는 랜덤 시드 같은 문구를 입력하는 창이 나오는데
나는 그냥 내 정체성 키워드를 입력했다! 뭔지는 시크릿임.

ptau 는 history 파일이라고 생각하면 쉬울 것 같다.

### 3. Second Contribution

`snarkjs powersoftau contribute pot12_0001.ptau pot12_0002.ptau --name="Second contribution" -v -e="some random text"`
=> pot12_0002.ptau 생성

### 4. 써드파티 프로그램을 이용해서 third contribution 을 생성한다.

1. `snarkjs powersoftau export challenge pot12_0002.ptau challenge_0003`
   => challenge_0003 생성
2. `snarkjs powersoftau challenge contribute bn128 challenge_0003 response_0003 -e="some random text"`
   => response_0003 생성
3. `snarkjs powersoftau import response pot12_0002.ptau response_0003 pot12_0003.ptau -n="Third contribution name"`
   => pot12_0003.ptau 생성

### 5. 지금까지 생성한 contribution verify!

MPC(multi-party computation)

`snarkjs powersoftau verify pot12_0003.ptau`
=> contribution 들 검증

### 6. random beacon

`snarkjs powersoftau beacon pot12_0003.ptau pot12_beacon.ptau 0102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f 10 -n="Final Beacon"`
=> pot12_beacon.ptau 생성

### 7. phase 2 준비?!

`snarkjs powersoftau prepare phase2 pot12_beacon.ptau pot12_final.ptau -v`
=> tauG1 => tauG2 => alphaTauG1 => betaTauG1 순으로 DEBUG

### 8. 마지막 ptau 검증!

circuit 서킷을 생성하기 전에 최종적으로 ptau 를 검증한다.

`snarkjs powersoftau verify pot12_final.ptau`

### 9. circom 파일 생성!

=> multiple 1000 으로 설정함. constraints 수를 의미한다.

cat <<EOT > circuit.circom
pragma circom 2.0.0;

template Multiplier(n) {
signal input a;
signal input b;
signal output c;

    signal int[n];

    int[0] <== a*a + b;
    for (var i=1; i<n; i++) {
    int[i] <== int[i-1]*int[i-1] + b;
    }

    c <== int[n-1];

}

component main = Multiplier(1000);
EOT

### 10. circom compile

`circom circuit.circom --r1cs --wasm --sym`
=> circuit.rlcs (the r1cs constraint system of the circuit in binary format)
circuit.sym (a symbols file required for debugging and printing the constraint system)
circuit.wasm (the wasm code to generate the witness)
파일을 생성

### >> 아니 근데 circom 도 설치 안한 상태에서 ㅋㅋㅋㅋ

`curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh`
`git clone https://github.com/iden3/circom.git`
`cargo build --release`
`cargo install --path circom`

이제 10번 된당!!

### 11. circuit 에 대한 상태 출력

`snarkjs r1cs info circuit.r1cs`

### 12. constraints 출력

`snarkjs r1cs print circuit.r1cs circuit.sym`

결과값은 아래와 같이 출력된다.

`snarkJS: [ 21888242871839275222246405745257275088548364400416034343698204186575808495616main.int[998] ] * [ main.int[998] ] - [ 21888242871839275222246405745257275088548364400416034343698204186575808495616main.c +main.b ] = 0`

### 13. export r1cs to json

`snarkjs r1cs export json circuit.r1cs circuit.r1cs.json; cat circuit.r1cs.json`
=> r1cs 를 json 파일로 export

### 14. witness 계산

`cat <<EOT > input.json {"a": 3, "b": 11} EOT`

EOT 가 vi 같은 명령어인가?
EOT 가 입력을 시작하고 닫을 때, 사용하는 것 같다.

=> circuit_js 폴더로 이동
`node generate_witness.js circuit.wasm ../input.json ../witness.wtns`
=> witness.wtns 생성

### 15. Setup

Plonk 나 Groth16 으로 r1cs ptau 를 계산하여 zeroknowledge key 인 zkey 를 생성하는데
이때 production 단계에서는 이 zkey 를 사용하면 안된다.

`zkey new` 명령어를 사용하여 zkey 가 생성되며 이때까지는 아직 contribution 이 없다.
그렇기 때문에 처음 circuit 에서 이 zkey 를 사용할 수 없다.

case #1. Plonk
`snarkjs plonk setup circuit.r1cs pot12_final.ptau circuit_final.zkey`
=> circuit_final.zkey 생성

case #2. Groth16
`snarkjs groth16 setup circuit.r1cs pot12_final.ptau circuit_0000.zkey`
=> circuit_0000.zkey

이때까지의 과정은 phase 1 이었고 앞으로는 power of tau 를 zkey 로 대체할 것이다.

### 16. phase 2 로 가즈아

`snarkjs zkey contribute circuit_0000.zkey circuit_0001.zkey --name="1st Contributor Name" -v`

랜덤값 입력하기!

0번째 circuit 의 상태를 1번째 circuit 에 담아 첫번째 contribute 를 실행한 것이다.
이때, `zkey create`와 `zkey contribute`의 차이점은 contribution 의 유무이다.
전자는 없고 후자는 있다!!

=> circuit_0001.zkey 생성

### 17. 두번째 contribute 가즈아

`snarkjs zkey contribute circuit_0001.zkey circuit_0002.zkey --name="Second contribution Name" -v -e="Another random entropy"`
=> circuit_0002.zkey 생성

### 18. 써드파티 프로그램을 사용하여 3번째 contribute 하기

`snarkjs zkey export bellman circuit_0002.zkey challenge_phase2_0003 snarkjs zkey bellman contribute bn128 challenge_phase2_0003 response_phase2_0003 -e="some random text" snarkjs zkey import bellman circuit_0002.zkey response_phase2_0003 circuit_0003.zkey -n="Third contribution name"`

이때 이용하는 써드파티 프로그램은 phase2-bn254
=> 오 이거 카이로 수업 때 나온 254비트 얘기같은데!!
https://github.com/kobigurk/phase2-bn254

=> circuit_0003.zkey, challenge_phase2_0003, response_phase2_0003 생성

### 19. 마지막 zkey 검증하기 (MPC)

`zkey verify` 명령어가 zkey "로" 검증하는 것이 아니라 zkey "를" 검증하는 것이었다!

`snarkjs zkey verify circuit.r1cs pot12_final.ptau circuit_0003.zkey`

=> ZKey Ok!

### 20. 랜덤 비콘 적용하기

`snarkjs zkey beacon circuit_0003.zkey circuit_final.zkey 0102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f 10 -n="Final Beacon phase2"`

`zkey beacon`는 랜덤 비콘으로 마지막 contribution 을 진행하기 위한 zkey 파일을 생성한다.

### 21. final zkey 검증하기

`snarkjs zkey verify circuit.r1cs pot12_final.ptau circuit_final.zkey`
=> Zkey Ok!

### 22. verification key 추출

circuit_final.zkey 에서 추출한다!
`snarkjs zkey export verificationkey circuit_final.zkey verification_key.json`
=> verification_key.json 생성

### 23. proof 생성!!!

PLONK 로 실행하면 Error: zkey file is not plonk 라고 에러 뜬다.
`snarkjs plonk prove circuit_final.zkey witness.wtns proof.json public.json`

Groth16 으로 실행하면 된다!
`snarkjs groth16 prove circuit_final.zkey witness.wtns proof.json public.json`
=> proof.json, public.json 생성

### 24. proof 검증

PLONK 는 일단 명령어만 적어두고
`snarkjs plonk verify verification_key.json public.json proof.json`

Groth16 으로 실행
`snarkjs groth16 verify verification_key.json public.json proof.json`
=> snarkJS: OK!

### 25. verifier 스마트컨트랙트로 생성

`snarkjs zkey export solidityverifier circuit_final.zkey verifier.sol`
=> verifier.sol 생성

### 26. 검증 트랜잭션 호출해보기!

`snarkjs zkey export soliditycalldata public.json proof.json`

`["0x0a110aa4423f719697bd4e496562788c4de532bcfd49e74a1ed33f9c44e784a1", "0x253c81d21e7bebc88e2a2eb6db3c84b5874c3d4729151cae1c9edd774f726a08"],[["0x1b5988625854a5ee1b50e28c9ca36206a6dee872df2a829d9bd79f674fc3a6be", "0x2818d8bf6c0ede3cac12cdf7ee87793b53574a4f270ace96880abf769c816f03"],["0x1918a5bbefa54a13336de9ba5f5cb040fe942357d388026c947a96264c08ac7f", "0x0b79c7dc370860cc5402ecde297088ab4c94690d2bc7aa643832e2b1321f6f44"]],["0x1f81b6b5f8cec63a999f41bd13d11fd472af791b02b70e79c71b3a573ee56f66", "0x1bed494a67e3f7572159a4f4672806c404e8fbc4c30daa5ee34b770df844c2c2"],["0x110d778eaf8b8ef7ac10f8ac239a14df0eb292a8d1b71340d527b26301a9ab08"]` 출력!

### 27. node 실행

구성환경은 다음과 같다.

1. `npm init`
2. `npm install snarkjs`
3. index.js 파일 생성
4. package.json scripts 에 npm start 로 `node index.js` 추가하기

이때, circuit_js 폴더 내에 있는 circuit.wasm 파일을 최상단으로 빼야 한다.
`npm start` 로 실행하면 proof 데이터 출력

### 28.client 실행

`cp node_modules/snarkjs/build/snarkjs.min.js .`
=> snark.min.js 생성
