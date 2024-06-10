참조 (&) vs 포인터 (*)

### 차이점

- 초기화 및 재할당
    - 참조: 선언과 동시에 초기화, 재할당 불가
      포인터: 선언 후 초기화 가능, 다른 주소값 할당 가능
- 함수 매개변수로 전달 시
    - 참조: 기존 변수의 메모리만 사용
      포인터: 스택 메모리에 주소값 생성

    ```cpp
    void square(int& num)
    {
    	num = num * num;
    }
    
    void square(int* num)
    {
    	*num = *num * *num;
    }
    
    int main()
    {
    	int number = 5;
    	int& numRef = number; // 1번
    	square(numRef);
    	cout << number << endl; //  >> 25
    
    	// 2번 
    	square(number); // int square(int&)
    	square(*number); // int square(int*)
    }
    
    ```

  1번 상황에서, numRef는 main 함수 스택 메모리에 올라가지 않는다.
  그러나 2번 상황에서는 모두 주소값을 전달하여 square 함수 스택에 올라간다.

- 이중 포인터
    - 참조: 불가
      포인터: 가능
- null value
    - 참조: 불가능
      포인터: 가능

요약:
참조는 문법적으로 reference by value와 같고, 기능적으로 reference by address와 같다.
포인터는 하나의 변수이고, 참조는 기존 변수의 별칭이라는 점에서 차이점이 발생한다.