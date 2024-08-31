# 제어할 수 없는 요소를 테스트하기

시간이나 난수 등 매 번 다르게 임의로 부여되는 값들은 개발자가 제어할 수 없습니다.

`isToday`라는 간단한 함수를 테스트 하는 예시를 들겠습니다.

```java
boolean isToday(LocalDateTime time) {
    LocalDate today = LocalDate.now();
    LocalDate givenDate = time.toLocalDate();
    return givenDate.equals(today);
}
```

테스트 하는 시점에 따라 today는 계속해서 바뀝니다.

해당 함수에 지금 당장 인자로 `2024-08-31`을 넣는다면 통과하겠지만, 내일부터는 영원히 실패하게 됩니다.

이렇듯 제어할 수 없는 시간이라는 요소는 테스트 하기 어렵다는 단점이 있으나,

ludo를 포함한 대부분의 application에서 `LocalDateTime`은 매우 빈번하게 사용됩니다.

처음에는 대부분의 코드베이스에 `LocalDateTime`이 사용되면 value를 직접 전달했지만 이런 문제점을 자각하고

`LocalDateTime`을 직접 전달하는 것 대신, `UtcDateTimePicker`라는 인터페이스를 만들어서 해당 객체로부터 현재 시각을 뽑도록 코드를 수정했습니다.

```java
boolean isToday(UtcDateTimePicker dateTimePicker) {
    LocalDate today = LocalDate.now();
    LocalDate givenDate = dateTimePicker.pick();
    return givenDate.equals(today); 
}
```

prefix가 `Utc`인 이유는 국제화를 고려하면 DB에 Utc+00 시간대를 저장하고 클라이언트에서 데이터를 가공하여 사용하도록 하는 것이 용량 절약 및 편의성 증대한다고 생각했기 때문입니다.

최근에는 내국인이 외국에서 접속하는 경우도 많기 때문에 내국인 타겟으로 서비스하는 어플리케이션도 시간대 정도는 신경 쓰면 좋을 것 같았습니다.

또한 기준이 명확하기에 프론트엔드 개발자와 협업 시에도 용이했습니다.

```java
public interface UtcDateTimePicker {
	LocalDateTime now();

	default LocalDateTime toMicroSeconds(LocalDateTime dateTime) {
		return dateTime.truncatedTo(ChronoUnit.MICROS);
	}
}
```

`UtcDateTimePicker` 인터페이스 명세는 매우 간단합니다.

`now`로 현재 시각을 뽑을 수 있습니다.

그런데 MySQL을 사용하던 도중 microseconds 이하 세부 데이터가 잘리는 현상이 발생하여 변환이 필요한 경우가 있었습니다.

이를 위해 `UtcDateTimePicker`로 뽑은 모든 `LocalDateTime`의 microseconds 이하를 제거하는 로직을 추가했습니다.(해당 로직 적용 후 문제가 해결되었습니다.)

```java
// CurrentUtcDateTimePicker
@Profile("!test")
@Component
public class CurrentUtcDateTimePicker implements UtcDateTimePicker {

	@Override
	public LocalDateTime now() {
		return LocalDateTime.now(ZoneOffset.UTC).truncatedTo(ChronoUnit.MICROS);
	}
}

// FixedUtcDateTimePicker
@Profile("test")
@Component
public class FixedUtcDateTimePicker implements UtcDateTimePicker {

    public static final LocalDateTime DEFAULT_FIXED_UTC_DATE_TIME = defaultFixedUtcDateTime();
    private LocalDateTime fixedUtcDateTime = DEFAULT_FIXED_UTC_DATE_TIME;

    @Override
    public LocalDateTime now() {
        return fixedUtcDateTime;
    }

    private static LocalDateTime defaultFixedUtcDateTime() {
        return LocalDateTime.of(2000, 1, 1, 0, 0, 0, 0);
    }

    // ...

}
```

그리고 2개의 구현체입니다.

개발 환경에서는 날짜를 통제할 필요가 없기 때문에 단순히 `LocalDateTime.now`를 호출하였고,

테스트 환경에서는 이를 항상 고정된 값으로 반환하도록 하였습니다.

덕분에 테스트 시에 현재 시각에 영향을 받지 않고 항상 일관된 결과를 도출할 수 있게 되었습니다.

profile이 test일 때는 자동으로 `FixedUtcDateTimePicker`를 spring context에 주입시켜서 별도의 설정 없이 구현체를 교환하여 편의성을 올렸습니다.

```java
// StudyService
@Service
@RequiredArgsConstructor
@Transactional
public class StudyService {

	private final StudyRepositoryImpl studyRepository;
	private final ApplicantRepositoryImpl applicantRepository;
	private final ParticipantRepositoryImpl participantRepository;
	private final UtcDateTimePicker utcDateTimePicker;
  // ...
public void attendance(final User user, final Long studyId) {
		final LocalDate now = utcDateTimePicker.now().toLocalDate(); // used
		final Study study = findStudyById(studyId);
		final Participant participant = findParticipantByStudyIdAndUserId(studyId, user.getId());
		if (!participant.isFirstAttendance()) {
			final LocalDate recentAttendanceDate = participant.getRecentAttendanceDate();
			isDuplicateAttendance(recentAttendanceDate, now); // 중복 출석 체크
		}
    // ...
```

이후 어플리케이션 전반에 걸쳐 `LocalDateTime.now()`가 호출되던 모든 코드를 `utcDateTimePicker.now()`로 교체하였습니다.

추후 microseconds 이하 데이터를 정상적으로 받을 수 있는 환경이 된다면 `truncate` 부분만 수정하면 모든 로직에 적용될 것이라 유지 보수 걱정도 덜게 되었습니다.
