# CVE-2024-4439

## Index
* [Index](#Index)
* [📝 Analysis](#-analysis)
    * [1. 개요](#1-개요)
    * [2. 발생 원인](#2-발생-원인)
    * [3. 패치 확인](#3-패치-확인)
* [🔫 POC](#-POC)
    * [1. 환경 셋팅(WordPress 설치 및 실행)](#1-환경-셋팅wordpress-설치-및-실행)
    * [2. XSS 페이로드 설정](#2-XSS-페이로드-설정)
    * [3. 트리거 준비](#3-트리거-준비)
    * [4. 익스플로잇](#4-익스플로잇)

## 📝 Analysis

### 1. 개요

`CVE-2024-4439`는 WordPress Core의 6.5.2 버전 미만에서 발생하는 저장형 XSS(Stored Cross-Site Scripting) 취약점입니다.

새 글 작성 시 아바타 블록을 추가하고, 해당 블록의 설정을 '사용자 프로필 링크' 및 '새 탭에서 열기'로 지정하면 사용자의 표시 이름이 출력됩니다. 이때 출력되는 값이 적절히 이스케이프 처리되지 않아 XSS 취약점이 발생합니다.

### 2. 발생 원인

아바타 블록의 사용자 표시 이름이 출력되는 코드는 `wp-includes/blocks/avatar.php` 에 다음과 같이 구현되어 있습니다.

> *WordPress Core 6.5 버전의 wp-includes/blocks/avatar.php
ref.* [https://github.com/WordPress/WordPress/blob/6.5/wp-includes/blocks/avatar.php#L52](https://github.com/WordPress/WordPress/blob/6.5/wp-includes/blocks/avatar.php#L52)
> 

```php
/* 생략 */
if ( isset( $attributes['isLink'] ) && $attributes['isLink'] ) {
    $label = '';
    if ( '_blank' === $attributes['linkTarget'] ) {
        // translators: %s is the Author name.
        $label = 'aria-label="' . sprintf( esc_attr__( '(%s author archive, opens in a new tab)' ), $author_name ) . '"';
    }
    // translators: %1$s: Author archive link. %2$s: Link target. %3$s Aria label. %4$s Avatar image.
    $avatar_block = sprintf( '<a href="%1$s" target="%2$s" %3$s class="wp-block-avatar__link">%4$s</a>', esc_url( get_author_posts_url( $author_id ) ), esc_attr( $attributes['linkTarget'] ), $label, $avatar_block );
}
/* 생략 */
```

위 코드에서 변수 `$label` 에 대입되는 코드는 다음과 같이 구현되어 있습니다.

```php
$label = 'aria-label="' . sprintf( esc_attr__( '(%s author archive, opens in a new tab)' ), $author_name ) . '"';
```

`sprintf` 함수는 서식 문자열 `%s` 에 변수를 삽입하는 역할을 하며, `esc_attr__` 함수는 인자로 전달된 문자열(`(%s author archive, opens in a new tab)`)을 이스케이프 처리하여 안전하게 출력합니다.

즉, 코드 흐름상 `esc_attr__` 함수가 먼저 호출되므로 이미 이스케이프 처리된 문자열에 사용자 표시 이름이 담긴 변수 `$author_name`가 서식 문자열 `%s` 에 포함됩니다.

따라서, 사용자 표시 이름에 악성 스크립트가 담겨져 있을 경우 이스케이프 처리가 되지 않으므로 XSS 취약점이 발생됩니다.

### 3. 패치 확인

WordPress Core 6.5.1 버전 부터는 취약점이 발생된 코드가 다음과 같이 변경되었습니다. ([diff](https://github.com/WordPress/WordPress/commit/66e36c2f18973585f973512bdfa91161d752d5f1#diff-f0927158422e81b2ab428447a5041985765dc99bc3757fe4edf55a0943e50c59R49))

![[https://github.com/WordPress/WordPress/commit/66e36c2f18973585f973512bdfa91161d752d5f1#diff-f0927158422e81b2ab428447a5041985765dc99bc3757fe4edf55a0943e50c59R49](https://github.com/WordPress/WordPress/commit/66e36c2f18973585f973512bdfa91161d752d5f1#diff-f0927158422e81b2ab428447a5041985765dc99bc3757fe4edf55a0943e50c59R49)](images/image-001.png)


변경된 코드를 살펴보면 다음과 같이 `sprintf` 함수를 먼저 호출하여 변수 `$author_name` 를 서식 문자열 `%s` 에 삽입한 뒤, `esc_attr` 함수의 인자로 전달하고 있습니다.

```php
$label = 'aria-label="' . esc_attr( sprintf( __( '(%s author archive, opens in a new tab)' ), $author_name ) ) . '"';
```

따라서, 사용자 표시 이름이 담긴 변수 `$author_name`에 악성 스크립트가 포함되더라도 함수 `esc_attr`에 의해 이스케이프 처리되므로 XSS 취약점이 더 이상 발생하지 않습니다.

## 🔫 POC

> 이미 WordPress가 설치되어 있는 경우 바로 ‘2. XSS 페이로드 설정’ 로 이동하셔도 됩니다 🙂

### 1. 환경 셋팅(WordPress 설치 및 실행)

`CVE-2024-4439` 에 영향을 받는 WordPress 6.5 버전을 내려받기 위해 아래의 명령어를 실행합니다.

```bash
# 저장소 클론
$ git clone -b CVE-2024-4439 https://github.com/DoTTak/Research-WordPress.git CVE-2024-4439

# 작업 경로 이동
$ cd CVE-2024-4439

# WordPress의 6.5 버전을 내려받기 위함
$ git submodule init 
$ git submodule update
```

이후, `docker-compose` 명령을 이용하여 WordPress를 실행합니다.

```bash
$ docker-compose up
```

그 다음 브라우저를 이용하여 `http://localhost:8080` 로 이동한 뒤, WordPress를 설치합니다.

> 아래 순서를 참고하여 진행 하시면 됩니다.
> 

1. WordPress 설치 - 언어 선택
    ![1. WordPress 설치 - 언어 선택](images/image-002.png)


2. WordPress 설치 - ‘시작합니다!’ 클릭
    ![2. WordPress 설치 - ‘시작합니다!’ 클릭](images/image-003.png)


3. WordPress 설치 - 데이터베이스 설정
    ![3. WordPress 설치 - 데이터베이스 설정](images/image-004.png)


4. WordPress 설치 - ‘설치 실행’ 클릭
    ![4. WordPress 설치 - ‘설치 실행’ 클릭](images/image-005.png)


5. WordPress 설치 - ‘사이트 정보’ 입력 후 ‘워드프레스 설치’ 클릭
    ![5. WordPress 설치 - ‘사이트 정보’ 입력 후 ‘워드프레스 설치’ 클릭](images/image-006.png)


6. WordPress 설치 - ‘로그인’ 클릭
    ![6. WordPress 설치 - ‘로그인’ 클릭](images/image-007.png)


### 2. XSS 페이로드 설정

이어서 나오는 로그인 페이지에서 조금 전 입력 한 사용자명과 비밀번호를 입력하여 로그인을 합니다. 이후, 나오는 대시보드 화면의 좌측 사이드바에서 `사용자 > 모든 사용자` 를 클릭 합니다.

![image.png](images/image-008.png)

그럼 방금 로그인한 사용자의 정보가 나오는데 해당 사용자의 ‘편집’으로 이동합니다.

![image.png](images/image-009.png)

이후 `이름` 부분에 ```" onmouseover='alert(`XSS`)' //``` 를 입력하고, `공개적으로 보일 이름` 에 방금 입력한 내용을 선택하고 프로필을 업데이트합니다.

![image.png](images/image-010.png)

### 3. 트리거 준비

`http://localhost:8080/wp-admin/post-new.php` 주소로 접속하여 다음과 같이 ‘새 글 추가’ 페이지로 이동합니다.

![스크린샷 2024-10-30 오전 12.05.13.png](images/image-011.png)

이후 다음과 같이 아바타 블록을 선택합니다.

![image.png](images/image-012.png)

아바타 블록이 생성되면 우측 블록 설정을 통해 ‘사용자 프로필로 가능 링크’ 와 ‘새 탭에서 열기’를 활성화하고 ‘사용자’란에 XSS 페이로드를 선택한 뒤 상단의 ‘공개’ 버튼을 클릭하여 해당 글을 공개합니다.

![image.png](images/image-013.png)

### 4. 익스플로잇

이후 작성된 글로 페이지를 이동한 뒤, 아바타 블록에 마우스를 가져다 대면(`onmouseover`) 다음과 같이 XSS 취약점이 발생되어 `alert` 이 발생됩니다.

![image.png](images/image-014.png)
