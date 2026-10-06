
## Как правильно готовить тесты
## Сравнение с текущим подходом
1. В статье основной акцент на правильном применении моков. 

Кроме библиотек для мокирования, иногда создаем эмуляторы внешних систем, которые соответствуют интерфейсу взаимодействия для оригинала и мы может протестировать наш код в интеграционных сценариях в части "эффектов более высокого уровня".

2. Основные несоответствия описываемому подходу в том, что в юнит тестах завязываемся на проверки того, что от мок вызывался тот или иной метод.

Как void методов в конце (отправка сообщения в очередь).
Так и в середине теле метода обращение к мок, который возвращает результаты. 

С void **варианты исправления**:
1) После успешной отправки возвращать true, а в случае неудачной ветки, если без исключения - false. Как чаще всего рефакторю такие участки.
Либо
2) Проверять реальную очередь - что выполненно целевое действие, но тогда это не мок.
Либо
3) Оставить как есть, с проверкой что у мок вызвался конкретный метод с нужным набором аргументов, а создаваемые им эффекты более высокого уровня тестировать на реальном классе, который подменяем моком.
Фактически так оно сейчас и работает.


## Вывод

Текущий подход не так уж и плох и соответствует 3-му _варианту исправления_.

## Примеры из pet проекта
```java
    @Test
  void checkGame_perfectRecallWithSevenLines_passesExpectedCountToSettings() {
    when(settingsService.resolveLinesCountForStart("user-key")).thenReturn(7);
    StartGameResponse started = gameService.startGame("user-key");
    List<UserLineRequest> userLines = started.lines().stream()
        .map(line -> new UserLineRequest(
            line.x(),
            line.y(),
            line.x() + line.dx(),
            line.y() + line.dy()))
        .toList();

    GameResult result = gameService.checkGame("user-key", new CheckRequest(started.rid(), userLines));

    assertThat(result.score()).isEqualTo(7.0);
    verify(settingsService).adjustAfterRound("user-key", 7, 0, 0);
  }

  @Test
  void checkGame_allCorrectLines_returnsFullScore() {
    when(settingsService.resolveLinesCountForStart("user-key")).thenReturn(5);
    StartGameResponse started = gameService.startGame("user-key");
    List<UserLineRequest> userLines = started.lines().stream()
        .map(line -> new UserLineRequest(
            line.x(),
            line.y(),
            line.x() + line.dx(),
            line.y() + line.dy()))
        .toList();

    GameResult result = gameService.checkGame("user-key", new CheckRequest(started.rid(), userLines));

    assertThat(result.score()).isEqualTo(5.0);
    assertThat(result.correct()).isEqualTo(5);
    assertThat(result.wrong()).isZero();
    assertThat(result.missed()).isZero();
    verify(historyRepository).save(eq("user-key"), eq(5.0), eq("+5 / -0.0 / -0"));
    verify(settingsService).adjustAfterRound("user-key", 5, 0, 0);
  }

  @Test
  void checkGame_reversedLineEndpoints_stillCountsAsCorrect() {
    when(settingsService.resolveLinesCountForStart("user-key")).thenReturn(5);
    StartGameResponse started = gameService.startGame("user-key");
    LineResponse line = started.lines().get(0);
    int x2 = line.x() + line.dx();
    int y2 = line.y() + line.dy();

    List<UserLineRequest> userLines = List.of(
        new UserLineRequest(x2, y2, line.x(), line.y()),
        new UserLineRequest(-1, -1, -2, -2),
        new UserLineRequest(-3, -3, -4, -4),
        new UserLineRequest(-5, -5, -6, -6),
        new UserLineRequest(-7, -7, -8, -8)
    );

    GameResult result = gameService.checkGame("user-key", new CheckRequest(started.rid(), userLines));

    assertThat(result.correct()).isEqualTo(1);
    assertThat(result.wrong()).isEqualTo(4);
    assertThat(result.score()).isEqualTo(-1.0);
    verify(settingsService).adjustAfterRound("user-key", 5, 4, 0);
  }

  @Test
  void checkGame_fewerLinesThanExpected_appliesMissedPenalty() {
    when(settingsService.resolveLinesCountForStart("user-key")).thenReturn(5);
    StartGameResponse started = gameService.startGame("user-key");
    LineResponse line = started.lines().get(0);
    List<UserLineRequest> userLines = List.of(
        new UserLineRequest(line.x(), line.y(), line.x() + line.dx(), line.y() + line.dy())
    );

    GameResult result = gameService.checkGame("user-key", new CheckRequest(started.rid(), userLines));

    assertThat(result.correct()).isEqualTo(1);
    assertThat(result.missed()).isEqualTo(4);
    assertThat(result.score()).isEqualTo(-3.0);
    verify(historyRepository).save(eq("user-key"), eq(-3.0), eq("+1 / -0.0 / -4"));
    verify(settingsService).adjustAfterRound("user-key", 5, 0, 4);
  }

  @Test
  void checkGame_unknownRoundId_returnsZeroResultAndSkipsSave() {
    GameResult result = gameService.checkGame(
        "user-key",
        new CheckRequest("unknown-round", List.of(new UserLineRequest(1, 1, 2, 2))));

    assertThat(result).isEqualTo(GameResult.sessionExpired());
    assertThat(result.expired()).isTrue();
    verify(historyRepository, never()).save(anyString(), anyDouble(), anyString());
    verify(settingsService, never()).adjustAfterRound(anyString(), anyInt(), anyInt(), anyInt());
  }
```

