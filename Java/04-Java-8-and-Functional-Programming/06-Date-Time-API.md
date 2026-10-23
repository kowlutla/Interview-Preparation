# Java Date-Time API

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21

The java.time API introduced in Java 8 provides immutable types for dates, times, instants, durations, periods, zones, and formatting. Correct production use depends on distinguishing a calendar date from a point on the timeline and making time-zone and daylight-saving assumptions explicit.

## 1. Date and time types

### Q1. Why was java.time introduced? *(Beginner)*

The older Date and Calendar APIs had mutable state, confusing month indexing, and awkward time-zone handling. java.time provides clearer immutable types aligned with ISO-8601 concepts and the Java Date-Time API design.

Prefer java.time for new code. Legacy APIs still appear in older libraries, so conversion boundaries may be necessary.

### Q2. What is the difference between LocalDate, LocalTime, and LocalDateTime? *(Beginner)*

LocalDate represents a calendar date without a time or zone. LocalTime represents a time of day without a date or zone. LocalDateTime combines date and time but still does not identify a unique instant on the global timeline because no time zone or offset is included.

Use local types for concepts that are genuinely local, such as a birthday or a store opening time. For events that must identify one moment globally, use Instant or an offset/zoned date-time.

### Q3. What is Instant? *(Intermediate)*

Instant represents a point on the timeline measured from the Unix epoch using seconds and nanoseconds. It is useful for event timestamps, ordering events, and storing globally meaningful moments.

An Instant does not preserve a user's time zone or preferred calendar display. Convert it to a ZonedDateTime or OffsetDateTime for presentation using an explicit zone or offset.

### Q4. What is the difference between OffsetDateTime and ZonedDateTime? *(Intermediate)*

OffsetDateTime includes a date-time and a fixed UTC offset. ZonedDateTime includes a date-time, an offset, and a time-zone ID backed by zone rules, including daylight-saving transitions.

A fixed offset such as +02:00 does not contain the future and historical daylight-saving rules of a region such as Europe/Athens. Use a ZoneId for regional civil-time rules and an Instant for an unambiguous timeline point.

### Q5. What is the difference between Duration and Period? *(Intermediate)*

Duration represents a time-based amount in seconds and nanoseconds. Period represents a calendar-based amount in years, months, and days. A month is not a fixed number of seconds, so the two types model different arithmetic.

Use Duration for elapsed timeouts and measurements. Use Period for calendar concepts such as adding one month to a date. Adding a month to January 31 requires calendar rules and may adjust to the last valid day.

### Q6. Why is LocalDateTime unsafe as a globally unique timestamp? *(Advanced)*

LocalDateTime has no zone or offset. During a daylight-saving overlap, a local time may correspond to two instants; during a gap, a local time may correspond to no valid instant in a particular zone.

For globally ordered events, store Instant and retain the relevant zone separately if business context requires it. Do not interpret a LocalDateTime as UTC unless the contract explicitly defines it that way.

## 2. Time zones and daylight saving

### Q7. What is ZoneId? *(Intermediate)*

ZoneId identifies a time zone, typically a region such as Europe/London or Asia/Kolkata, with rules supplied by the time-zone database. A ZoneOffset is a fixed offset from UTC and does not by itself encode regional rule changes.

Use region-based zones for appointments, local schedules, and business opening hours. Time-zone rules can change due to political decisions, so systems may need updated tzdb data.

### Q8. What happens during a daylight-saving gap or overlap? *(Advanced)*

When clocks move forward, some local times do not exist. When clocks move backward, some local times occur twice. java.time APIs define resolution behavior and provide methods to inspect valid offsets and adjust local date-times.

For scheduling, decide whether the business rule follows the earlier offset, later offset, or shifts a nonexistent time. Test transitions explicitly; assuming every local date-time maps to exactly one instant causes production scheduling bugs.

### Q9. Why should systems store timestamps in UTC? *(Advanced)*

Storing an Instant or a clearly defined UTC timestamp gives a stable global timeline and avoids ambiguity caused by local daylight-saving transitions. Local display can be derived later using the user's or business's zone.

UTC storage does not remove the need to retain the original business zone for recurring appointments or local deadlines. A daily meeting at 9 AM local time is a zone-based schedule, not simply a repeated fixed UTC instant.

### Q10. How do you calculate elapsed time correctly? *(Intermediate)*

Use Instant or Duration for elapsed time. LocalDateTime arithmetic can be misleading around zone transitions because it has no zone semantics. System.currentTimeMillis and nanoTime also have different purposes: wall-clock timestamp versus elapsed-time measurement.

For measuring duration within a process, System.nanoTime is generally preferred because it is designed for elapsed-time measurement and is not a wall clock. Do not compare nanoTime values across processes or treat them as epoch timestamps.

## 3. Parsing, formatting and persistence

### Q11. How should date-time strings be parsed? *(Intermediate)*

Use the appropriate formatter and temporal type for the input contract. ISO-8601 strings with offsets can be parsed into OffsetDateTime or Instant; date-only strings fit LocalDate. Custom patterns should be explicit and tested.

Do not parse a local timestamp and silently assume a time zone. Validation should reject or deliberately resolve invalid dates and ambiguous times according to the domain policy.

### Q12. What is DateTimeFormatter, and is it thread-safe? *(Intermediate)*

DateTimeFormatter is immutable and thread-safe, making it suitable for reuse. This differs from the legacy SimpleDateFormat, which is mutable and not thread-safe without coordination.

Prefer predefined ISO formatters where possible. For custom patterns, define locale and zone explicitly when output must be deterministic across servers.

### Q13. How should timestamps be stored in a database? *(Senior)*

Choose a database type and convention that preserve the required semantics. For event timestamps, store a globally meaningful instant or UTC-normalized value, and document precision and conversion rules. For business schedules, preserve the local date/time and zone if future zone-rule changes matter.

Database timestamp types vary in whether they preserve offsets or normalize values. Verify the behavior of the database, JDBC driver, ORM mapping, and session time zone together. Test round-trips around daylight-saving transitions.

### Q14. How do you handle clock skew in distributed systems? *(Senior)*

Different hosts' wall clocks can differ due to synchronization delays, configuration, or clock adjustments. Timestamps are useful for diagnostics and approximate ordering but should not be treated as a perfect global total order.

Use monotonic clocks for local elapsed-time measurements, correlation IDs and traces for request causality, and database or coordination primitives for correctness-critical ordering. If business logic depends on expiry, define allowed skew and use a trusted time source consistently.

### Q15. What is the difference between plusDays and plusHours around daylight saving? *(Advanced)*

On a ZonedDateTime, plusDays follows the local calendar timeline and can preserve local clock time across a daylight-saving transition, so the elapsed duration may be 23 or 25 hours. plusHours adds a fixed number of timeline hours.

Choose based on intent: “same local time tomorrow” is calendar arithmetic; “exactly 24 elapsed hours later” is duration arithmetic. Test both behaviors around transitions.

### Q16. How should recurring schedules be modeled? *(Senior)*

Store the recurrence rule, local time, relevant ZoneId, start/end constraints, and a clear policy for gaps and overlaps. A recurring schedule should generally be expanded using regional zone rules rather than by repeatedly adding a fixed 24-hour duration.

When time-zone rules change, decide whether future occurrences should follow updated rules or preserve previously calculated instants. This is a business decision that should be explicit in the data model.

## Official references

1. [Java SE 21 API — java.time package](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/time/package-summary.html)
2. [Java SE 21 API — Instant](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/time/Instant.html)
3. [Java SE 21 API — ZonedDateTime](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/time/ZonedDateTime.html)
4. [Java SE 21 API — DateTimeFormatter](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/time/format/DateTimeFormatter.html)
5. [JEP 150 — Date & Time API](https://openjdk.org/jeps/150)

## Chapter checklist

- [ ] Distinguish local civil time, offset date-time, zoned time, and Instant.
- [ ] Use Duration for elapsed time and Period for calendar arithmetic.
- [ ] Define daylight-saving gap/overlap policies for schedules.
- [ ] Persist time zone context when future local-time rules matter.
- [ ] Use DateTimeFormatter and test database round-trips.
