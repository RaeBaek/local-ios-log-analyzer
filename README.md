# Local iOS Log Analyzer

로컬 개발 중 수집한 iOS 앱 로그에서 **첫 실패, 근거가 되는 원문과 줄 번호, 다음 확인 지점**을 보여주는 터미널 도구입니다. 로그 파일이나 파이프에서 입력을 받으며, 아래 형식으로 출력된 로그는 프로젝트에 관계없이 분석할 수 있습니다. 다음 확인 항목을 고를 때는 필요에 따라 Mac의 Ollama를 사용합니다.

## 로그 작성 형식

분석기에 전달하는 텍스트는 **이벤트 하나당 한 줄**로 작성합니다. 각 줄에 `시간 [레벨] [컴포넌트] 메시지`를 넣습니다.

```text
10:00:00.120 [INFO] [HTTPClient] GET /search started
10:00:15.121 [ERROR] [HTTPClient] GET /search failed: NSURLErrorDomain Code=-1001 (timed out)
```

- **시간**은 `HH:mm:ss` 또는 `HH:mm:ss.SSS` 형식입니다. 분석기는 파일에 기록된 순서대로 첫 실패를 찾습니다.
- **레벨**은 대문자로 적습니다. `ERROR`, `FAULT`, `CRITICAL`, `FATAL`을 실패로 분류합니다. 정상 시작·완료나 검색 결과 0건 같은 이벤트에는 `INFO`를 사용할 수 있습니다.
- **컴포넌트**에는 `HTTPClient`나 `MovieRepository`처럼 로그가 발생한 위치를 적습니다. **메시지**에는 동작, 결과, 오류 코드처럼 디버깅에 필요한 정보를 남기고 인증 헤더·토큰·개인정보는 제외합니다.

앱에 기존 로거가 있다면 그 로거에서 이 형식으로 출력하거나, 수집할 때 로그를 이 형식으로 변환하면 됩니다. Swift `Logger`를 쓰는 경우 시간·레벨·카테고리가 메시지와 별도로 표시될 수 있으니, 분석기에 넘기는 텍스트에도 필요한 정보가 들어 있는지 확인하세요.

## 준비

이 저장소를 내려받고 루트에서 실행합니다. 기존 `local-llm-lab`의 가상환경에 `openai`가 이미 설치되어 있다면 그 환경을 사용해도 됩니다.

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install openai
```

## 파일 또는 파이프로 분석

```bash
python analyze_log.py samples/normal.log
python analyze_log.py samples/decoding.log
cat samples/normal.log | python analyze_log.py -
```

`normal.log`는 `명시적 실패 없음`을 표시합니다. `decoding.log`는 `movie.title`의 타입 불일치를 첫 실패로 표시합니다. `-`는 파일 대신 표준 입력에서 로그를 읽는다는 뜻입니다.

`samples/timeout.log`와 `samples/decoding.log`의 다음 확인 항목을 모델로 분류하려면 Mac에서 Ollama를 실행하고 `qwen2.5:7b`를 내려받으세요. 모델 요청은 로컬 Ollama의 `127.0.0.1:11434`로 전송됩니다.

## 연동 예시 · Showcase 시뮬레이터

분석기에는 Showcase의 `HomeNetwork` 로그를 원래 형식 그대로 읽는 규칙도 있습니다. 아래 명령은 Showcase의 subsystem과 category로 수집 대상을 필터링합니다. 다른 프로젝트에서 시뮬레이터 로그를 파이프로 연결할 때는 해당 프로젝트의 로그를 위 공통 형식으로 변환해 넘기면 됩니다.

Mac에서 `HomeNetwork` 로그 변경이 적용된 Showcase 앱과 iOS 시뮬레이터를 준비하세요. 앱 쪽 변경 범위는 [Showcase 이슈 #38](https://github.com/RaeBaek/Showcase/issues/38)을 참고하세요. 시뮬레이터를 먼저 켠 다음 터미널에서 아래 명령을 실행합니다.

```bash
xcrun simctl spawn booted log stream --level info --style compact --color always \
  --predicate 'subsystem == "com.raehoon.Showcase.app" AND category == "HomeNetwork"' \
  | tee /dev/tty \
  | python analyze_log.py -
```

Xcode에서 앱을 실행하고 첫 화면의 요청이 끝나면 터미널에서 `Control+C`를 누르세요. 수집 중에는 색상이 적용된 로그가 보이고, 수집을 마치면 분석 결과가 표시됩니다. `-`는 로그를 파이프로 받는다는 뜻입니다.

`samples/showcase-success.log`와 `samples/showcase-offline.log`로 같은 입력 형식을 확인할 수 있습니다. 성공 샘플과 `URLError code -1009`가 기록된 오프라인 샘플은 Python 규칙으로 처리합니다.

## 분석 범위

분석기는 첫 실패가 기록된 줄을 기준으로 결과를 출력합니다. `명시적 실패 없음`은 수집한 구간에 지원 형식의 실패 줄이 없다는 뜻입니다. 입력은 UTF-8이며 실행 한 번에 최대 12,000바이트까지 받습니다.
