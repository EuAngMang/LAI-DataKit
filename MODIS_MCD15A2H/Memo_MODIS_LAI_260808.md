2026.08.08. 토요일
MODIS LAI - MCD15A2H 에 대한 메모

# 기본 정보
- 한 파일 = 8일 컴포짓 하나 (8일 대표값)
- 전지구, $0.05^{\circ} \times 0.05^{\circ}$, 3,200 $\times$ 7,200 grids
- 식생군(biome) 축 포함

# LAI 변수
총 6개로 구성되어 있음
- `lai_main_land`
  : 주 복사전달(Radiative Transfer, RT) 알고리즘 (SCF_QC = 0, 1) & LandSea = 육지
- `lai_main_nonland`
  : 주 복사전달 알고리즘 & 비육지(해안/내수/해양)
  : 혼합화소라 물 비율만큼 LAI 가 희석되어 육지보다 LAI 값이 낮은 편
- `lai_backup`
  : 백업 경험적 NDVI 알고리즘 (SCF_QC = 2, 3)
- `lai_main_land_biome`
  : lai_main_land 의 biome 별 분리판
- `lai_main_nonland_biome`
  : lai_main_nonland 의 biome 별 분리판
- `lai_backup_biome`
  : lai_backup 의 biome 별 분리판

여섯 변수 모두 `int16, scale_factor=0.001, add_offset=0.0, _Fillvalue=-32768` 이 적용되어 있음

# LAI 외 변수
각 LAI 변수마다 3개씩 짝이 되는 화소 개수 관련 변수가 있음

통합 LAI 변수와 짝이 되는 화소 수 변수
- n_main_land
- n_main_nonland
- n_backup

biome 별로 분리된 변수와 짝이 되는 화소 수 변수
- n_main_land_biome
- n_main_nonland_biome
- n_backup_biome

셀에서 LAI 가 없는 화소를 센 변수
- `n_nonveg`
  : 비식생 분류코드(249-254: 도시/불모지/물) 화소 수
  : 결측이 아닌 화소에 대해 토지피복 분류 중 비식생 화소 수를 센 것
- `n_missing`
  : 미산출(255 또는 SCF_QC = 4) 화소 수
  : 영구 결측인 화소 수를 센 것
- `n_total`
  : 원자료로부터 0.05도 셀에 기여한 화소 수
  : 고위도에서 0인 것은 격자 성질에 의한 불가피한 특징
  : $main\_land + main\_nonland + backup + nonveg + missing == n\_total$

# LAI 계산 시 주의점
## 서로 다른 LAI 변수로 계산 시 기여 화소 수 고려
기본적으로 LAI 값은 셀마다 각 변수별로 평균이 되어 있는 상태임. lai_main_land 는 그 셀에서 main_land 화소들만의 평균, lai_backup 은 backup 화소들만의 평균이 계산되어 있는 것.

그렇기 때문에 한 셀에서 lai_main_land 와 lai_backup 둘 모두를 고려한 LAI 를 구하기 위해서는 각 변수에 참여한 화소 수를 고려해줘야 함. lai_main_land 와 lai_backup 의 화소 수는 0.05도 내에서 차지하는 비율이 다르기 때문.

$LAI = {\sum_X( lai_X \times n_X) \over \sum_X(n_X)}$
$\ \ \ \ \ \ \ \ ={lai\_main\_land \times n\_main\_land + lai\_backup \times n\_backup \over n\_main\_land + n\_backup}$

분자: 각 LAI 변수에 속한 원자료(500m 해상도) 화소들의 LAI 를 전부 더한 값
분모: 계산에 참여한 화소들의 개수

X 는 선택한 LAI 변수를 의미함
X $\in$ { main_land, main_nonland, backup }

수치적으로 예시를 들면 아래와 같음
어떤 셀에 원자료 화소가 100개가 들어 있었고, 다음과 같이 들어 있다고 가정.
- main_land: 80개, 평균 LAI=3.0
- main_nonland: 15개, 평균 LAI=1.2 (해안 혼합화소라 육지보다 낮음)
- backup: 5개, 평균 LAI=0.5

main_land 단독으로 사용할 경우,
$LAI=3.0$

main_land 와 main_nonland 를 사용할 경우(해안을 포함하는 경우),
$LAI={(80 \times 3.0 + 15 \times 1.2) \over 80 + 15}$
$\ \ \ \ \ \ \ \ = {240 + 18 \over 95} = 2.716$

main_land, main_nonland, backup 모두 사용할 경우,
$LAI = {80 \times 3.0 + 15 \times 1.2 + 5 \times 0.5 \over 80 + 15 + 5}$
$\ \ \ \ \ \ \ \ = {240 + 18 + 2.5 \over 100} = 2.605$

만약 세 변수의 값을 단순히 평균하면 안 됨
$LAI = {3.0 + 1.2 + 0.5 \over 3} = 1.567$

셀 안에 포함된 화소 수가 변수마다 다름에도 불구하고 단순 평균에서는 동등하게 비교를 한 것이기 때문. 이는 공간 평균을 수행할 때에도 동일하게 적용됨.

참고 사항
1. 이 데이터는 등면적 누적($500m \times 500m$=1화소)으로 만들어졌기 때문에 화소 개수가 곧 면적을 의미함. 따라서 위도 가중치를 주지 않아도 됨.
2. LAI 값은 해당 변수의 화소 수가 0인 곳에서 NaN 이기 때문에 NaN $\times$ 0 = NaN 으로 인해 계산이 잘못될 수 있음. 따라서 `nansum`으로 누적하는 것을 권함.
3. 비식생(nonveg)을 0으로 세는 정의(물이나 도시는 LAI가 0)를 쓰고자 한다면 분모가 n_total - n_missing 이 됨. n_total 을 단순히 사용하면 결측을 나지로 인식하여 세는 꼴이 됨. 즉, 화소 수에서 결측을 우선 제거해야 한다는 것. 따라서 수식으로 쓰면 다음과 같음
   $LAI\_nonveg\_as\_zero = LAI \times {\sum_X(n_X) \over n_total - n_missing}$
4. main_land + backup 은 사실 비대칭 조합임. lai_backup 은 LandSea 로 걸러져 있지 않은 상태이기 때문에 비육지 화소를 품고 계산된 상태임. 하지만 main_land 는 main_nonland 로 분리되어 있음. 대칭으로 계산하고자 한다면 main_land + main_nonland + backup 이거나 main_land 단독으로 사용해야 하고, 추후 lai_backup 에 대한 육지/해양 분리가 수행될 예정.

# 필요 지식
## 복사 전달 알고리즘 (Radiative Transfer, RT)
MODIS LAI/FPAR 은 산출 당시 적용된 알고리즘이 둘임
- 주 알고리즘
  : 3차원 복사전달 모델로 만든 LUT(Look-Up Table)를 반사도에 대해 역산해서 LAI 를 구하는 방식으로, 물리 기반. (Shabanov et al., 2000)
- 백업 알고리즘
  : 주 알고리즘이 실패했을 때 사용하는 NDVI-LAI 경험 관계식으로 LAI 를 구하는 방식으로, 물리 기반이 아닌 경험 기반.

파일 내의 QC 값으로는 SCF_QC 에 기록되어 있으며, 0과 1은 주 알고리즘의 성공을, 2와 3은 백업 알고리즘으로 넘어갔음을 의미함.

## 원자료에서의 QC 정보
원자료에서 QC 는 서로 독립인 필드로 구성되어 있음
- FparLai_QC
  : SCF_QC 필드에서 알고리즘 정보가 기록되어 있음 (0, 1 = 주, 2, 3 = 백업)
- FparExtra_QC
  : LandSea 필드에서 육지/비육지가 기록되어 있음
  : 비육지는 크게 해안(coast), 내수면(inland water), 해양(ocean)으로 나눌 수 있는데, 해안이 대부분이며, 내수면은 호수나 큰 강을 의미함. 해양은 FparExtra_QC의 LandSea 필드에 거의 없는 편으로, 순수 해양은 이미 비식생 코드 254로 빠져 n_nonveg 로 들어가기 때문임.

## 혼합화소
MODIS LAI (MCD15A2H)의 원자료는 500m $\times$ 500m 해상도로, 넓이로는 0.25 $km^2$ 임. 화소 하나가 지표에서 한 종류만 덮고 있으리라는 보장이 없음. 해안의 어떤 화소는 육지 60%, 바다 40% 가 함께 들어 있을 수 있는 것.

인공위성의 센서는 그 화소를 하나의 반사도 값으로 측정함. 육지 몫과 바다 몫을 따로 주지 않고, 면적으로 섞인 한 덩이로 인식함. 이것이 혼합화소(mixed pixel)의 정체임.

주 알고리즘은 그 한 덩어리 반사도를 받아 화소 전체가 하나의 식생 군락이라고 가정하고 복사전달 LUT 를 역산함. 즉, 화소 안에 물이 섞여 있다고 고려하지 않는 것.

물은 근적외 영역에서 매우 어두움(반사가 덜 됨). 반면, 식생은 근적외를 강하게 반사함. 따라서 물이 섞인 화소에서는 반사도가 상대적으로 낮게 측정되고, 알고리즘은 그것을 "잎이 적은 군락"으로 인식할 뿐임. 인공위성이 받아들이는 반사도는 아래와 같이 구성되어 있다 볼 수 있음.
(단, 역산이 비선형이기 때문에 정비례하지 않음 주의)

관측 반사도 = f_land $\times$ 식생 반사도 + f_water $\times$ 물 반사도

위성의 센서는 이렇게 관측된 반사도를 육지로부터 왔다고 가정하여 받아들일 뿐임. 그래서 결국 혼합화소가 포함된 셀은 육지만 포함된 셀보다 LAI 가 낮은 편임.