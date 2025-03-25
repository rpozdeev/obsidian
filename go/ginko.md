# Тестирование с ginko и gomega

## Установка Ginkgo и Gomega

```bash
go get github.com/onsi/ginkgo/v2
go install github.com/onsi/ginkgo/v2/ginkgo

go get -u github.com/onsi/gomega/...
```

Чтобы выбрать конкретную версию:
```bash
go get github.com/onsi/ginkgo/v2@v2.m.p
go install github.com/onsi/ginkgo/v2/ginkgo
```
## Структура тестов с Ginkgo

```go
package mypackage_test

import (
    "testing"
    "github.com/onsi/ginkgo/v2"
    "github.com/onsi/gomega"
)

func TestMyPackage(t *testing.T) {
    gomega.RegisterFailHandler(ginkgo.Fail)
    ginkgo.RunSpecs(t, "MyPackage Suite")
}

var _ = ginkgo.Describe("SomeFunction", func() {
    ginkgo.It("should return true when input is positive", func() {
        result := SomeFunction(1)
        gomega.Expect(result).To(gomega.BeTrue())
    })

    ginkgo.It("should return false when input is negative", func() {
        result := SomeFunction(-1)
        gomega.Expect(result).To(gomega.BeFalse())
    })
})
```

- **ginkgo.Describe** — описывает группу тестов, например, поведение какой-то функции.
- **ginkgo.It** — описывает отдельный тест.
- **gomega.Expect** — утверждение, проверяющее результат.

## Основные утверждения Gomega

Gomega предоставляет множество утверждений, например:

- gomega.Expect(x).To(gomega.Equal(expected)) — проверяет, что x равно expected.
- gomega.Expect(x).To(gomega.BeTrue()) — проверяет, что x истинно.
- gomega.Expect(x).NotTo(gomega.BeNil()) — проверяет, что x не nil.
- gomega.Expect(x).To(gomega.BeEmpty()) — проверяет, что x пустой.
- gomega.Expect(x).To(gomega.ContainSubstring(substring)) — проверяет, что строка содержит подстроку.

## Запуск тестов

```bash
ginkgo
```

Также можно запускать тесты через стандартную команду go test, но с Ginkgo ты получишь более удобный вывод:
```bash
go test -v
```
