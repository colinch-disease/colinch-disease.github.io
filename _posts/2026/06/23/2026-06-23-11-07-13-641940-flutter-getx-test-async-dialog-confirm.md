---
layout: post
title: "Flutter GetX Controller 테스트 - 비동기 다이얼로그 처리하는 법"
description: "GetX Controller 테스트에서 삭제 확인 다이얼로그처럼 비동기 UI 상호작용이 필요한 케이스를 콜백 주입 패턴으로 해결하는 방법을 실제 코드로 정리했다."
date: 2026-06-23
tags: [Flutter, UnitTest, GetX, CleanArchitecture, Dart]
comments: true
share: true
---

# Flutter GetX Controller 테스트 - 비동기 다이얼로그 처리하는 법

![코드 테스트](https://images.unsplash.com/photo-1516116216624-53e697fedbea?w=800&q=80)

[이전 편에서 Fake Repository를 만들고 Controller 기본 테스트를 잡았다.]({% post_url 2026-06-23-10-00-00-629595-flutter-unit-test-repository-layer-start %}) 거기서 끝에 예고한 대로, MemberManagementController를 테스트하다가 막혔다. 멤버 삭제 버튼 → 확인 다이얼로그 → "확인" 탭 → API 호출. 이 흐름을 Unit Test에서 어떻게 재현하나.

## Get.defaultDialog를 테스트에서 쓸 수 없다

처음 Controller 코드는 이랬다.

```dart
Future<void> removeMember(String memberId) async {
  Get.defaultDialog(
    title: '멤버 삭제',
    middleText: '정말 삭제하시겠습니까?',
    onConfirm: () async {
      Get.back();
      await _repository.removeMember(memberId: memberId);
    },
  );
}
```

`flutter test`로 돌리면 바로 터진다. `GetMaterialApp` 없이 `Get.defaultDialog`를 호출하면 위젯 트리가 없다는 오류다. Widget Test로 환경을 만들면 `Get.put()` 초기화가 꼬이고, `pumpWidget`에 `GetMaterialApp`을 넣으면 또 다른 문제가 생겼다.

두 시간 삽질하고 나서 결론이 나왔다. "Controller가 다이얼로그를 직접 띄우면 안 된다."

## 콜백 주입 패턴

"사용자가 확인했는지 물어보는 동작"을 `Future<bool> Function()` 타입으로 외부에서 주입받는다.

```dart
class MemberManagementController extends GetxController {
  MemberManagementController({
    required MemberRepository repository,
    Future<bool> Function()? confirmRemove,
  })  : _repository = repository,
        _confirmRemove = confirmRemove ??
            (() async {
              final result = await Get.defaultDialog<bool>(
                title: '멤버 삭제',
                middleText: '이 멤버를 공간에서 삭제합니다.',
                onConfirm: () => Get.back(result: true),
                onCancel: () => Get.back(result: false),
              );
              return result ?? false;
            });

  final MemberRepository _repository;
  final Future<bool> Function() _confirmRemove;
  final isLoading = false.obs;
  final members = <Member>[].obs;

  Future<void> removeMember(String memberId) async {
    final confirmed = await _confirmRemove();
    if (!confirmed) return;

    isLoading.value = true;
    final result = await _repository.removeMember(memberId: memberId);
    isLoading.value = false;

    result.fold(
      onSuccess: (_) => members.removeWhere((m) => m.id == memberId),
      onFailure: (msg) => Get.snackbar('오류', msg),
    );
  }
}
```

기본값이 실제 다이얼로그를 띄우는 함수라서, 앱 코드에서 `MemberManagementController(repository: repo)`처럼 그냥 쓰면 된다. 프로덕션 동작은 그대로다.

## Controller 테스트

```dart
void main() {
  late FakeMemberRepository fakeRepo;
  late MemberManagementController controller;

  void setup({bool userConfirms = true}) {
    fakeRepo = FakeMemberRepository();
    Get.put<MemberRepository>(fakeRepo);
    controller = Get.put(
      MemberManagementController(
        repository: fakeRepo,
        confirmRemove: () async => userConfirms,
      ),
    );
  }

  tearDown(Get.reset);

  group('removeMember', () {
    test('취소하면 Repository를 호출하지 않는다', () async {
      setup(userConfirms: false);
      fakeRepo.setMembers([Member(id: 'm1', name: '홍길동')]);

      await controller.removeMember('m1');

      expect(fakeRepo.removeCallCount, 0);
      expect(controller.members.length, 1);
    });

    test('확인하면 멤버가 목록에서 제거된다', () async {
      setup(userConfirms: true);
      fakeRepo.setMembers([Member(id: 'm1', name: '홍길동')]);
      controller.members.assignAll(fakeRepo.members);

      await controller.removeMember('m1');

      expect(fakeRepo.removeCallCount, 1);
      expect(controller.members, isEmpty);
    });

    test('서버 오류 시 members 목록이 유지된다', () async {
      setup(userConfirms: true);
      fakeRepo.setMembers([Member(id: 'm1', name: '홍길동')]);
      fakeRepo.shouldFail = true;
      controller.members.assignAll(fakeRepo.members);

      await controller.removeMember('m1');

      expect(controller.members.length, 1);
    });
  });
}
```

`setup()` 헬퍼에 `userConfirms` 파라미터 하나로 "사용자가 확인했다/취소했다" 시나리오가 명확하게 표현된다. 테스트 읽는 사람이 다이얼로그 동작을 추론할 필요가 없다.

## Fake Repository에 호출 카운터 추가

Mockito 없이 Fake 구현체에 `removeCallCount` 필드를 직접 달았다. 이 정도 검증 수준에서 Mockito를 끌어오는 건 오버킬이었다.

```dart
class FakeMemberRepository implements MemberRepository {
  final List<Member> _members = [];
  int removeCallCount = 0;
  bool shouldFail = false;

  void setMembers(List<Member> members) {
    _members
      ..clear()
      ..addAll(members);
  }

  List<Member> get members => List.from(_members);

  @override
  Future<Result<void>> removeMember({required String memberId}) async {
    removeCallCount++;
    if (shouldFail) return Result.failure('서버 오류');
    _members.removeWhere((m) => m.id == memberId);
    return Result.success(null);
  }
}
```

## 처음에 Completer로 시도했다가 포기한 이유

콜백 주입 전에 `Completer<bool>`을 Controller 안에 두고 테스트에서 외부에서 `complete(true)`를 호출하는 방식을 시도했다. `await controller.removeMember()`가 Completer를 기다리는 동안 테스트 코드가 멈추니까, completer를 먼저 complete하고 나서 `await`를 걸어야 했다. 순서가 보장이 안 됐다. 비동기 타이밍 맞추다가 테스트가 flaky해지는 전형적인 패턴이었다.

콜백 주입은 Controller 입장에서 "확인 함수를 호출하고 결과를 기다린다"는 인터페이스가 동일하면서, 테스트에서는 그냥 `() async => true`를 넘기면 끝난다.

---

다음 편은 `flutter_blue_plus` BLE 관련 테스트다. `FlutterBluePlus` 클래스가 static 메서드 위주라서 의존성 격리를 다른 방식으로 해야 한다.
