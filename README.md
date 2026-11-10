"""
AI처럼 계산하는 계산기 (표준 라이브러리만 사용)

AI(LLM)가 계산하는 방식을 단순화해서 흉내 냅니다.
  1) 토큰화      : 숫자를 3자리 단위 조각으로 읽는다
  2) 직관 모드   : 답을 한 자리씩 '확률적으로' 예측한다 (틀릴 수 있음)
  3) 단계별 모드 : 풀이 과정을 적어 가며 정확하게 계산한다

실행: python ai_calculator.py
"""

import random

PLACES = ["일", "십", "백", "천", "만", "십만", "백만", "천만"]


# ---------- 1) 토큰화 ----------
def tokenize(n: int) -> list[str]:
    s = str(n)
    return [s[i:i + 3] for i in range(0, len(s), 3)]


# ---------- 정답 계산 ----------
def exact(a: int, b: int, op: str):
    if op == "+":
        return a + b
    if op == "-":
        return a - b
    if op == "*":
        return a * b
    if op == "/":
        return None if b == 0 else a // b
    raise ValueError("지원하지 않는 연산자입니다. (+ - * /)")


# ---------- 2) 직관 모드 ----------
def digit_distribution(correct: int, length: int, pos: int) -> list[float]:
    """자릿수가 많고 뒤쪽 자리일수록 틀릴 확률이 커지도록 단순화한 모형."""
    err = min(0.6, 0.02 * length ** 1.5 * (0.6 + 0.4 * pos / max(1, length - 1)))
    probs = [0.0] * 10
    probs[correct] = 1 - err
    near = [(correct + 1) % 10, (correct - 1) % 10]
    others = [d for d in range(10) if d != correct and d not in near]
    for d in near:
        probs[d] = err * 0.3
    for d in others:
        probs[d] = err * 0.4 / len(others)
    return probs


def intuition_mode(answer: int) -> None:
    print("\n[② 직관 모드] 한 자리씩 확률로 답을 찍습니다")
    sign = "-" if answer < 0 else ""
    s = str(abs(answer))
    guess = ""
    for i, ch in enumerate(s):
        probs = digit_distribution(int(ch), len(s), i)
        g = random.choices(range(10), weights=probs)[0]
        guess += str(g)
        top = sorted(range(10), key=lambda d: probs[d], reverse=True)[:3]
        cand = ", ".join(f"{d}:{probs[d] * 100:.0f}%" for d in top)
        mark = "O" if g == int(ch) else "X"
        print(f"  {i + 1}번째 자리 → 선택 {g} [{mark}]  (후보: {cand})")
    result = int(sign + guess)
    if result == answer:
        print(f"  ✅ AI의 직관 답: {result:,} (정답)")
    else:
        print(f"  ❌ AI의 직관 답: {result:,} / 실제 정답: {answer:,}")
        print("     한 자리만 틀려도 전체가 틀립니다. 풀이 과정을 적으면 정확해져요.")


# ---------- 3) 단계별 풀이 모드 ----------
def add_steps(a: int, b: int) -> list[str]:
    A, B = str(a)[::-1], str(b)[::-1]
    steps, carry = [], 0
    for i in range(max(len(A), len(B))):
        x = int(A[i]) if i < len(A) else 0
        y = int(B[i]) if i < len(B) else 0
        s = x + y + carry
        extra = f" + 올림 {carry}" if carry else ""
        steps.append(f"{PLACES[i]}의 자리: {x} + {y}{extra} = {s} → {s % 10} 쓰고 "
                     f"{'1 올림' if s >= 10 else '올림 없음'}")
        carry = 1 if s >= 10 else 0
    if carry:
        steps.append("남은 올림 1 → 맨 앞에 씀")
    return steps


def sub_steps(a: int, b: int) -> list[str]:
    steps = []
    if a < b:
        steps.append("앞 수가 더 작음 → 큰 수에서 작은 수를 빼고 마지막에 − 부호를 붙임")
        a, b = b, a
    A, B = str(a)[::-1], str(b)[::-1]
    borrow = 0
    for i in range(len(A)):
        x = int(A[i]) - borrow
        y = int(B[i]) if i < len(B) else 0
        if x < y:
            steps.append(f"{PLACES[i]}의 자리: {x} < {y} → 10 빌려서 {x + 10} - {y} = {x + 10 - y}")
            borrow = 1
        else:
            steps.append(f"{PLACES[i]}의 자리: {x} - {y} = {x - y}")
            borrow = 0
    return steps


def mul_steps(a: int, b: int) -> list[str]:
    steps, total = [], 0
    for i, d in enumerate(reversed(str(b))):
        part = a * int(d) * 10 ** i
        total += part
        shift = f" (×{10 ** i:,})" if i else ""
        steps.append(f"{a} × {d}{shift} = {part:,}")
    steps.append(f"부분 곱을 모두 더하면 → {total:,}")
    return steps


def div_steps(a: int, b: int) -> list[str]:
    if b == 0:
        return ["0으로는 나눌 수 없습니다"]
    steps, rem, q = [], 0, ""
    for ch in str(a):
        cur = rem * 10 + int(ch)
        d = cur // b
        steps.append(f"내려쓴 수 {cur} ÷ {b} = 몫 {d}, 나머지 {cur - d * b}")
        q += str(d)
        rem = cur - d * b
    steps.append(f"몫 = {int(q):,}, 나머지 = {rem}")
    return steps


def step_mode(a: int, b: int, op: str, answer) -> None:
    print("\n[③ 단계별 풀이 모드] 생각을 적어 가며 계산합니다")
    fn = {"+": add_steps, "-": sub_steps, "*": mul_steps, "/": div_steps}[op]
    for line in fn(a, b):
        print("  ·", line)
    if answer is not None:
        print(f"  ✅ 최종 답: {answer:,}")


# ---------- 메인 ----------
def parse(text: str):
    """'478 + 256' 형태의 입력을 (a, op, b)로 변환"""
    text = text.replace("×", "*").replace("÷", "/").replace("x", "*")
    for op in "+-*/":
        # 맨 앞 부호와 구분하기 위해 1번째 글자 이후에서 찾음
        idx = text.find(op, 1)
        if idx != -1:
            a, b = text[:idx].strip(), text[idx + 1:].strip()
            if a.isdigit() and b.isdigit():
                return int(a), op, int(b)
    raise ValueError("형식이 올바르지 않습니다. 예) 478 + 256")


def main() -> None:
    print("=== AI처럼 계산하는 계산기 ===")
    print("예) 478 + 256 / 1234 * 56   (종료: q)")
    while True:
        text = input("\n식 입력 > ").strip()
        if text.lower() in ("q", "quit", "exit"):
            print("종료합니다.")
            break
        try:
            a, op, b = parse(text)
        except ValueError as e:
            print("  ⚠", e)
            continue

        print(f"\n[① 토큰화] {tokenize(a)} {op} {tokenize(b)}")
        print("  AI는 숫자를 한 덩어리가 아니라 이런 조각으로 읽습니다.")

        answer = exact(a, b, op)
        if answer is None:
            print("  ⚠ 0으로는 나눌 수 없습니다.")
            continue
        intuition_mode(answer)
        step_mode(a, b, op, answer)


if __name__ == "__main__":
    main()
