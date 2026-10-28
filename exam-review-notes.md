# Session 5 — Exam Review Notes
Compiled from our walkthrough: concepts + the code we built together (Kigali Weather guide + your Study Room Booking System for Part 2). This is a study reference, not a project to hand in — you wrote the Booking/UsageRecord pieces yourself; make sure you can still explain every line below out loud.

---

## 1. Future, async, await

A `Future<T>` is "a `T`, eventually" — the *ticket*, not the food.
`async` marks a function as allowed to pause. `await` is *where* it pauses — only that function waits; nothing else freezes.

```dart
Future<String> orderBrochette() async {
  await Future.delayed(Duration(seconds: 3));
  return 'Your brochette is ready';
}
```

Forgetting `await` gets you `Instance of 'Future'` — the ticket, not the food.

---

## 2. Calling an API + turning JSON into a Dart object

```dart
class Weather {
  final double temperature;
  final int humidity;
  final double wind;
  final int code;

  Weather({required this.temperature, required this.humidity, required this.wind, required this.code});

  factory Weather.fromJson(Map<String, dynamic> json) => Weather(
    temperature: (json['temperature_2m'] as num).toDouble(),
    humidity: (json['relative_humidity_2m'] as num).toInt(),
    wind: (json['wind_speed_10m'] as num).toDouble(),
    code: (json['weather_code'] as num).toInt(),
  );
}

Future<Weather> fetchWeather(City city) async {
  final url = Uri.parse(
    'https://api.open-meteo.com/v1/forecast'
    '?latitude=${city.latitude}&longitude=${city.longitude}'
    '&current=temperature_2m,relative_humidity_2m,wind_speed_10m,weather_code',
  );
  final response = await http.get(url);
  if (response.statusCode != 200) {
    throw Exception('The weather server answered ${response.statusCode}');
  }
  final data = jsonDecode(response.body) as Map<String, dynamic>;
  return Weather.fromJson(data['current'] as Map<String, dynamic>);
}
```

**Why `as num` then `.toDouble()`/`.toInt()`, only for external API data?**
JSON doesn't distinguish `25` from `25.0`. The *server's* number might arrive either way depending on the actual value, so read as `num` (either kind) then convert. This does **not** apply to data only your own `toJson` ever writes (e.g. `numberOfOccupants`) — that always round-trips as the same type, so a plain `as int` is correct and *more* honest there.

**Why check `statusCode` before parsing the body?**
A non-200 response might not even be JSON (could be an HTML error page). Checking first lets you `throw` your own clear exception instead of getting a confusing `FormatException` from `jsonDecode`.

**Why does one `catch` block handle "bad status" *and* "no internet at all"?**
With no internet, `http.get()` itself throws (a `SocketException`) before any response exists — execution jumps straight from `try` into `catch`, skipping the status check entirely. One generic `catch (e)` covers both failure modes with the same Retry UI.

---

## 3. The three-state UI pattern (loading / error / data)

```dart
bool loading = true;
String? error;
Weather? weather;

Future<void> load() async {
  setState(() { loading = true; error = null; });
  try {
    final w = await fetchWeather(selectedCity);
    setState(() { weather = w; loading = false; });
  } catch (e) {
    setState(() { error = 'Could not load the weather. Check your internet connection.'; loading = false; });
  }
}

// build(): if (loading) show spinner; else if (error != null) show message + Retry; else show data.
```

`build()` never fetches — it only *reads* state and decides which of the three to show. Fetching is a side effect, so it only happens explicitly: `initState()`, `onSelected`, and the Retry button's `onPressed`. `build()` can re-run many times for unrelated reasons (animations, parent rebuilds), so it must stay side-effect-free.

---

## 4. Switching between options (cities)

```dart
class City {
  final String name;
  final double latitude;
  final double longitude;
  City(this.name, this.latitude, this.longitude);
}

final cities = [
  City('Kigali', -1.9441, 30.0619),
  City('Huye', -2.5967, 29.7394),
  City('Musanze', -1.4998, 29.6344),
  City('Rubavu', -1.6792, 29.2585),
];
// ChoiceChip onSelected: setState(() => selectedCity = city); load();
```

Updating `selectedCity` only triggers a rebuild — it does not fetch. `onSelected` must call `load()` itself.

---

## 5. Remembering a single choice (`shared_preferences`)

```dart
final prefs = await SharedPreferences.getInstance();   // itself async — touches disk
await prefs.setString('city', selectedCity.name);      // save AFTER fetch succeeds
// ...
final saved = prefs.getString('city');                 // 'Musanze', or null the first time
```

`shared_preferences` only stores `String`/`int`/`bool`/`double`/`List<String>` — never a whole object.

Save **after** success, not on tap: if you saved immediately and the fetch failed, next launch would try to restore a city you never actually confirmed worked — and (see §6) there'd be no cached weather for it either.

---

## 6. Caching one object for offline use (`toJson` / `jsonEncode`)

```dart
await prefs.setString('weather_${selectedCity.name}', jsonEncode(weather.toJson()));
// on failure:
final cached = prefs.getString('weather_${selectedCity.name}');
if (cached != null) weather = Weather.fromJson(jsonDecode(cached));
```

Chain to memorize: object → `toJson()` → `Map` → `jsonEncode` → `String` (to store). Reverse to load: `String` → `jsonDecode` → `Map` → `fromJson()` → object.

---

## 7. Saving/loading a *list* of your own objects (Part 2 generalization)

Your `Booking` and `UsageRecord` models (Study Room Booking System):

```dart
enum BookingStatus { requested, approved, used, cancelled }

class Booking {
  final String id;
  final String studentName;
  final String supervisorName;
  final DateTime date;
  final int startingHour;
  final String purpose;
  final BookingStatus status;

  Booking({required this.id, required this.studentName, required this.supervisorName,
    required this.date, required this.startingHour, required this.purpose, required this.status});

  Map<String, dynamic> toJson() => {
    'id': id, 'studentName': studentName, 'supervisorName': supervisorName,
    'date': date.toIso8601String(), 'startingHour': startingHour,
    'purpose': purpose, 'status': status.name,
  };

  factory Booking.fromJson(Map<String, dynamic> json) => Booking(
    id: json['id'] as String,
    studentName: json['studentName'] as String,
    supervisorName: json['supervisorName'] as String,
    date: DateTime.parse(json['date'] as String),
    startingHour: json['startingHour'] as int,
    purpose: json['purpose'] as String,
    status: BookingStatus.values.byName(json['status'] as String),
  );

  Booking copyWith({String? id, String? studentName, String? supervisorName, DateTime? date,
      int? startingHour, String? purpose, BookingStatus? status}) => Booking(
    id: id ?? this.id,
    studentName: studentName ?? this.studentName,
    supervisorName: supervisorName ?? this.supervisorName,
    date: date ?? this.date,
    startingHour: startingHour ?? this.startingHour,
    purpose: purpose ?? this.purpose,
    status: status ?? this.status,
  );
}

class UsageRecord {
  final String id;
  final String bookingId;                 // links back to its Booking
  final List<String> equipmentUsed;
  final String remarks;
  final int numberOfOccupants;
  final DateTime recordedDate;

  UsageRecord({required this.id, required this.bookingId, required this.equipmentUsed,
    required this.remarks, required this.numberOfOccupants, required this.recordedDate});

  Map<String, dynamic> toJson() => {
    'id': id, 'bookingId': bookingId, 'equipmentUsed': equipmentUsed,
    'remarks': remarks, 'numberOfOccupants': numberOfOccupants,
    'recordedDate': recordedDate.toIso8601String(),
  };

  factory UsageRecord.fromJson(Map<String, dynamic> json) => UsageRecord(
    id: json['id'] as String,
    bookingId: json['bookingId'] as String,
    equipmentUsed: List<String>.from(json['equipmentUsed'] ?? []),
    remarks: json['remarks'] as String? ?? '',
    numberOfOccupants: json['numberOfOccupants'] as int,
    recordedDate: DateTime.parse(json['recordedDate'] as String),
  );
}
```

Enums aren't JSON types either: store `status.name` (a `String`), read back with `BookingStatus.values.byName(...)`.

**Saving/loading the whole list** — the step-8 pattern generalized:

```dart
const _bookingsKey = 'bookings_key';

Future<void> saveBookings(List<Booking> bookings) async {
  final prefs = await SharedPreferences.getInstance();
  await prefs.setString(_bookingsKey, jsonEncode(bookings.map((b) => b.toJson()).toList()));
}

Future<List<Booking>> loadBookings() async {
  final prefs = await SharedPreferences.getInstance();
  final saved = prefs.getString(_bookingsKey);
  if (saved == null) return [];
  final jsonList = jsonDecode(saved) as List;
  return jsonList.map((j) => Booking.fromJson(j as Map<String, dynamic>)).toList();
}
```

The chain: `List<Booking>` → `.map(toJson)` → `List<Map>` → `jsonEncode` → one `String`. Reversed exactly to load. Same four steps for *any* list of *any* model — `saveUsageRecords`/`loadUsageRecords` are identical with the model and key string swapped.

**Cast explicitly at the boundary**: write `Model.fromJson(j as Map<String, dynamic>)`, not `Model.fromJson(j)` — `j` is `dynamic` either way, but the explicit cast fails immediately with a clear error if an item is ever malformed, instead of failing confusingly later inside `fromJson`.

**Linking two lists**: filter by the foreign key, don't duplicate data:

```dart
List<UsageRecord> usageFor(String bookingId, List<UsageRecord> all) =>
    all.where((r) => r.bookingId == bookingId).toList();
```

---

## 8. Updating one item inside a saved list

Two valid approaches:

```dart
// A) Functional — rebuild the whole list
Future<void> markBookingAsUsed(String bookingId) async {
  final bookings = await loadBookings();
  final updated = bookings.map((b) => b.id == bookingId ? b.copyWith(status: BookingStatus.used) : b).toList();
  await saveBookings(updated);
}

// B) Imperative — mutate one slot
final index = bookings.indexWhere((b) => b.id == bookingId);
if (index != -1) bookings[index] = bookings[index].copyWith(status: BookingStatus.used);
```

Both are correct. (A) rebuilds the list; (B) mutates one slot and needs an explicit "not found" guard (`-1`) that (A) doesn't, since `.map` silently no-ops on no match.
