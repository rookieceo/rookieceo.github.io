---
published: true
title: '코딩테스트 뒤풀이'
date: '2022-08-17 12:00:00 +0900'
categories: etc
last_modified_at: '2022-08-17 12:00:00 +0900'
tags:
  - coding-test
  - java8
  - algorithm
toc_sticky: true
toc: true
---

# 코딩 테스트 뒤풀이

- 모기업에서 진행하는 코딩테스트를 봤다. 
- 회사 일 하면서 이런 걸 해볼 일이 많이 있을까?
- 실제 직접 구현하기보다는 좋은 예제를 찾고 조합/응용하여 적용하는 나로써는 구글링으로 문제해결하는 데만 익숙한 터라 그다지 잘하지는 않지만 해보기로 했다.
- codility 라는 사이트에서 진행하였고 한 시간 반동안 총 3개의 문제를 풀어야했다.
- 3문제 중 두 문제만 풀었고 두번째 문제는 우왕좌왕하다 확실하게 푼 것 같지도 않다.
- 시간 제약이 없다는 가정하에 문제를 다시한번 보니 아쉬웠다.
- 첫 번째 문제에 대한 뒤풀이로 리팩토링하여 기록해본다.
- 문제는 아래와 같다.

```text

--- 원문 ---
Write a function:
class Solution { public int solution(int N); }
which, given an integer N, returns the smallest integer that is greater than N and the sum of whose digits is equal to the sum of the digits of N.

Examples:
1. Given N = 28, your function should return 37. The sum of the digits of 28 is equal to 2 + 8 = 10. The subsequent numbers are (with the sum of their digits in brackets): 29 (11), 30 (3), 31 (4), 32 (5), 33 (6), 34 (7), 35 (8), 36 (9) and 37 (10). 37 is the smallest number bigger than 28 whose digits add up to 10.
2. Given N = 734, your function should return 743. The sum of the digits of 734 and 743 are equal 7 + 3 + 4 = 7 + 4 + 3 = 14. No other integer between 735 and 742 adds up to 14.
3. Given N = 1990, your function should return 2089. The sum of the digits of both numbers is equal to 19 and there is no other integer between them with the same sum of digits.
4. Given N = 1000, your function should return 10000. The sum of the digits of both numbers is equal to 1 and there is no other integer between them with the same sum of digits.

Assume that:
N is an integer within the range [1..50,000].
In your solution, focus on correctness. The performance of your solution will not be the focus of the assessment.

--- 번역 ---
함수를 작성하십시오 :
클래스 솔루션 | 공개 int 솔루션 (int N); }
정수 N이 주어지면 N보다 큰 가장 작은 정수와 그 자릿수의 합이 N의 자릿수의 합과 같습니다.

예를 들자면
1. N = 28이 주어지면 함수는 37을 반환해야 합니다. 28의 자릿수의 합은 2 + 8 = 10과 같다. 후속 숫자는 29 (11), 30 (3), 31 (4), 32 (5), 33 (6), 34 (7), 35 (8), 36 (9) 및 37 (10)입니다. 37은 숫자가 10이 되는 28보다 큰 가장 작은 숫자다.
2. N=734가 주어졌을 때, 함수는 743을 반환해야 한다. 734와 743의 자릿수의 합은 7 + 3 + 4 = 7 + 4 + 3 = 14이다. 735와 742 사이의 다른 정수는 14까지 합산되지 않습니다.
3. N = 1990이 주어지면 함수는 2089를 반환해야 합니다. 두 숫자의 자릿수의 합은 19와 같으며 동일한 자릿수의 합계가 있는 다른 정수는 없습니다.
4. N = 1000이 주어지면 함수는 10000을 반환해야 합니다. 두 숫자의 숫자의 합은 1과 같으며 숫자의 합이 같은 다른 정수는 없습니다.

N은 [1.50,000] 범위 내의 정수라고 가정합니다.
당신의 해결책은 정확성에 초점을 맞추는 것이다.솔루션의 성능이 평가의 초점이 되지 않습니다.

```

- 문제 핵심
  - 정수형 변수 N의 각 자리수 합을 구하기(1..50,000)
  - N의 자리수 합과 동일한 N 이후 다음 숫자를 구하기
  

- 당시 작성했던 풀이
```java
class SolutionA {

  public int solution(int N) {
    // write your code in Java SE 8
    // 50000보다 클 경우 50000 리턴
    if(N > 50000){
      return 50000;
    }

    // 최대 루프 값 - 50000자리수의 다음 수로 해당하는 결과
    int maximumLoopCount = 100004;

    int nSumValue = getIntWordSum(N);
    int result = 0;

    // N은 최대 50000까지의 수이므로 50000 에 대한 다음 숫자를 최대값으로 정함
    for(int j = N + 1; j <= maximumLoopCount; j++) {
      int nextSumValue = getIntWordSum(j);
      if(nSumValue == nextSumValue){
        result = j;
        break;
      }
    }
    return result;
  }  
  
  public static int getIntWordSum(int N){
    String nValue = "" + N;
    int nSize = nValue.length();
    int nSum = 0;

    for(int i = 0; i < nSize; i++){ 
      nSum = nSum + Integer.parseInt(Character.toString(nValue.charAt(i)));
    }

    return nSum;
  }  
}
```
 - 아쉬운게 많았다.
   - 첫번째, getIntWordSumA 함수 : 자리수의 합 구하기
     - 뒤풀이를 찾아보니 두 종류가 있더라
       - 모드 연산을 통한 방법(아래와 같이)
       ```java
          int solution(int n) { 
            int sum = 0; 
            while(n > 0){ 
                sum += n %10;
                n /= 10; // 오른쪽이동  
            } 
            return sum; 
          }  
       ```
       - 문자열로 변환한 방법
         - 내가 적용한 방법
       - 난 전자인 모드연산을 활용한 방법은 가독성 측면에서 별로라고 생각한다.
       - 그래서 두번째 방법으로 하되, 반복문을 Stream으로 바꿀 수 있을 것 같다.
   - 두번째, 반복문의 제약, 방식
     - N이 50,000까지의 수 것을 조건으로 50,000의 결과값을 MaxLoopCount로 지정한 것
     - for -> do while 이 좋을 것 같다.
   - 세번째 변수 범위에 대한 유효성
     - 0 혹은 -1 등의 정수값이 들어왔을 때에는 유효성 검증을 더 해야할 것 같다.
   
 - 리팩토링 한 코드는 아래와 같다. 더 좋은 방법이 있겠지만 여기까지

 ```java
class SolutionB{
  public static int solution(int N) {
    // write your code in Java SE 8

    // 유효성 검증
    // 1보다 작거나 50000보다 클 경우에는 요청값 그대로 리턴
    if(N < 1 || N > 50000){
      return N;
    }
    
    // 자리수 합계 구하기
    int nSumValue = getIntWordSum(N);
    int result;

    int j = N + 1;
    int nextSumValue;

    // 1씩 증가하며 다음 수의 합계와 같은지 비교
    // 같으면 반복문 종료
    do{
      nextSumValue = getIntWordSum(j);
      result = j;
      j++;
    }while (nSumValue != nextSumValue);

    return result;
  }

  
  public static int getIntWordSum(int N){
    // 스트림을 활용
    // 정수 -> Array 스트림으로 변환
    // mapToInt를 통해 정수형으로 변환, sum 값을 출력
    return Stream.of(("" + N).split("")).mapToInt(Integer::valueOf).sum();
  }
}

  
```