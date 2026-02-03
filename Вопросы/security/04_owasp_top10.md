## Вопрос: OWASP Top 10 (кратко)

## Простой ответ
- OWASP Top 10 — список самых критичных рисков веб-приложений. Актуальная версия: 2025 (A01–A10).

## Ответ
OWASP Top 10 — стандартный «скелет» для AppSec-программы и чек-лист рисков, который помогает не забыть базовые классы уязвимостей.

**Актуальный список (2025):**
1. A01 Broken Access Control.
2. A02 Security Misconfiguration.
3. A03 Software Supply Chain Failures.
4. A04 Cryptographic Failures.
5. A05 Injection.
6. A06 Insecure Design.
7. A07 Authentication Failures.
8. A08 Software or Data Integrity Failures.
9. A09 Security Logging and Alerting Failures.
10. A10 Mishandling of Exceptional Conditions.

**Важно:** OWASP периодически обновляет список; в практике ещё часто встречаются материалы по Top 10:2021, но ориентироваться стоит на 2025.

## Примеры

1. A01: доступ к `/admin` без проверки роли.
2. A05: открытая админка и дефолтные пароли.
3. A03: компрометированная зависимость в цепочке поставки.

## Доп. теория

1. Top 10 — ориентир, а не полный список уязвимостей.
2. Класс уязвимости включает множество конкретных багов.
