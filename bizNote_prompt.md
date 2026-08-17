# BizNote — 비즈니스 노트앱 Claude Code 개발 프롬프트

---

## 📋 프로젝트 개요

iPhone과 iPad를 지원하는 비즈니스 전용 노트 앱 **"BizNote"**를 Swift + SwiftUI로 개발한다.
업무일지, 회의록, 전시회 방문 기록 등 카테고리별 노트 관리와 명함 스캔 자동 입력, Excel 추출 기능을 핵심으로 한다.

---

## 🛠️ 기술 스택 및 개발 환경

- **언어**: Swift 5.10+
- **UI 프레임워크**: SwiftUI (iOS 17+ 타겟)
- **데이터 저장**: SwiftData (CoreData 대체, iOS 17+)
- **동기화**: CloudKit 연동 (SwiftData + iCloud)
- **카메라/스캔**: VisionKit (`VNDocumentCameraViewController`)
- **OCR**: Vision framework (`VNRecognizeTextRequest`) — 한국어, 영어, 중국어 지원
- **Excel 출력**: `XLSXF` 패키지 (Swift Package Manager)
  - SPM URL: `https://github.com/jlowgren/xlsxf` 또는 대안으로 CSV → 시스템 공유 시트 활용
- **다국어**: Localizable.xcstrings (한국어 기본, 영어 지원)
- **최소 지원 버전**: iOS/iPadOS 17.0
- **Xcode**: 16.0+
- **아키텍처**: MVVM + Repository 패턴

### 패키지 매니저 설정 (Package.swift 또는 Xcode SPM)
```
dependencies:
  - XLSXF: https://github.com/jlowgren/xlsxf (Excel 생성)
```

---

## 🗂️ 앱 구조 및 파일 트리

```
BizNote/
├── BizNoteApp.swift                  # 앱 엔트리포인트, SwiftData 컨테이너 설정
├── ContentView.swift                 # 루트 뷰 (사이드바 + NavigationSplitView)
│
├── Models/
│   ├── NoteCategory.swift            # 카테고리 열거형 및 SwiftData 모델
│   ├── Note.swift                    # 노트 SwiftData 모델
│   ├── BusinessCard.swift            # 명함 데이터 SwiftData 모델
│   └── NoteTemplate.swift            # 카테고리별 템플릿 구조체
│
├── ViewModels/
│   ├── NoteListViewModel.swift       # 노트 목록 상태 및 필터링
│   ├── NoteDetailViewModel.swift     # 노트 상세/편집 상태
│   ├── BusinessCardViewModel.swift   # 명함 스캔 및 OCR 처리
│   └── ExportViewModel.swift         # Excel/CSV 추출 로직
│
├── Views/
│   ├── Main/
│   │   ├── SidebarView.swift         # iPad 사이드바 (카테고리 목록)
│   │   ├── NoteListView.swift        # 노트 목록
│   │   └── EmptyStateView.swift      # 빈 상태 안내 뷰
│   │
│   ├── Note/
│   │   ├── NoteDetailView.swift      # 노트 상세 보기/편집
│   │   ├── NoteEditorView.swift      # 노트 작성 에디터
│   │   ├── CategoryPickerView.swift  # 카테고리 선택
│   │   └── Templates/
│   │       ├── WorkLogTemplate.swift       # 업무일지 템플릿 뷰
│   │       ├── MeetingMinutesTemplate.swift # 회의록 템플릿 뷰
│   │       └── ExhibitionTemplate.swift    # 전시회 템플릿 뷰
│   │
│   ├── BusinessCard/
│   │   ├── BusinessCardScanView.swift      # 명함 스캔 카메라 뷰
│   │   ├── BusinessCardResultView.swift    # 스캔 결과 확인/수정 뷰
│   │   ├── BusinessCardListView.swift      # 명함 목록 뷰
│   │   └── BusinessCardRowView.swift       # 명함 목록 행 컴포넌트
│   │
│   ├── Export/
│   │   ├── ExportView.swift          # 내보내기 옵션 시트
│   │   └── ExportFilterView.swift    # 내보내기 필터 (날짜, 카테고리)
│   │
│   └── Settings/
│       └── SettingsView.swift        # 설정 (언어, 테마 등)
│
├── Services/
│   ├── OCRService.swift              # Vision OCR 처리 서비스
│   ├── BusinessCardParser.swift      # OCR 텍스트 → 명함 필드 파싱
│   └── ExcelExportService.swift      # Excel/CSV 파일 생성 서비스
│
├── Utilities/
│   ├── DateFormatter+Extensions.swift
│   ├── Color+Extensions.swift
│   └── View+Extensions.swift
│
└── Resources/
    ├── Localizable.xcstrings         # 한국어/영어 다국어 문자열
    └── Assets.xcassets               # 앱 아이콘, 색상 팔레트
```

---

## 📦 데이터 모델 상세 정의

### 1. NoteCategory (카테고리)

```swift
// Models/NoteCategory.swift
import Foundation

enum NoteCategory: String, Codable, CaseIterable, Identifiable {
    case workLog       = "work_log"       // 업무일지
    case meetingMinutes = "meeting_minutes" // 회의록
    case exhibition    = "exhibition"     // 전시회
    case custom        = "custom"         // 사용자 정의

    var id: String { rawValue }

    var localizedName: String {
        switch self {
        case .workLog:        return String(localized: "category.workLog")
        case .meetingMinutes: return String(localized: "category.meetingMinutes")
        case .exhibition:     return String(localized: "category.exhibition")
        case .custom:         return String(localized: "category.custom")
        }
    }

    var systemIconName: String {
        switch self {
        case .workLog:        return "briefcase.fill"
        case .meetingMinutes: return "person.2.fill"
        case .exhibition:     return "building.columns.fill"
        case .custom:         return "folder.fill"
        }
    }

    var accentColor: String {
        switch self {
        case .workLog:        return "AccentBlue"
        case .meetingMinutes: return "AccentGreen"
        case .exhibition:     return "AccentOrange"
        case .custom:         return "AccentPurple"
        }
    }
}
```

### 2. Note (노트 메인 모델)

```swift
// Models/Note.swift
import SwiftData
import Foundation

@Model
final class Note {
    var id: UUID
    var title: String
    var category: NoteCategory
    var createdAt: Date
    var updatedAt: Date

    // 공통 필드
    var content: String          // 자유 메모 영역
    var tags: [String]           // 태그 배열

    // 카테고리별 구조화 필드 (JSON String으로 저장)
    var templateData: String     // JSON encoded NoteTemplateData

    // 연결된 명함
    @Relationship(deleteRule: .nullify) var businessCards: [BusinessCard]

    // 첨부파일 (이미지 경로 배열)
    var attachmentPaths: [String]

    // 즐겨찾기
    var isFavorite: Bool

    init(
        title: String = "",
        category: NoteCategory = .workLog,
        content: String = ""
    ) {
        self.id = UUID()
        self.title = title
        self.category = category
        self.content = content
        self.createdAt = Date()
        self.updatedAt = Date()
        self.tags = []
        self.templateData = "{}"
        self.businessCards = []
        self.attachmentPaths = []
        self.isFavorite = false
    }
}
```

### 3. BusinessCard (명함 모델)

```swift
// Models/BusinessCard.swift
import SwiftData
import Foundation

@Model
final class BusinessCard {
    var id: UUID
    var createdAt: Date
    var scannedLanguage: String  // "ko", "en", "zh"

    // 기본 정보
    var name: String             // 이름
    var namePhonetic: String     // 발음 (한자 이름 등)
    var company: String          // 회사명
    var department: String       // 부서
    var jobTitle: String         // 직함
    var email: String
    var phone: String            // 휴대폰
    var officePhone: String      // 사무실 전화
    var fax: String
    var address: String          // 주소
    var website: String
    var memo: String             // 추가 메모

    // 원본 스캔 이미지 경로
    var imagePath: String

    // 연결된 노트 (역방향)
    var note: Note?

    init() {
        self.id = UUID()
        self.createdAt = Date()
        self.scannedLanguage = "ko"
        self.name = ""
        self.namePhonetic = ""
        self.company = ""
        self.department = ""
        self.jobTitle = ""
        self.email = ""
        self.phone = ""
        self.officePhone = ""
        self.fax = ""
        self.address = ""
        self.website = ""
        self.memo = ""
        self.imagePath = ""
    }
}
```

### 4. 카테고리별 템플릿 데이터 구조

```swift
// Models/NoteTemplate.swift
import Foundation

// 업무일지 템플릿
struct WorkLogTemplateData: Codable {
    var date: Date = Date()
    var workItems: [WorkItem] = []        // 업무 항목 목록
    var achievements: String = ""          // 주요 성과
    var issues: String = ""               // 이슈/문제점
    var nextTodos: String = ""            // 다음 할 일

    struct WorkItem: Codable, Identifiable {
        var id: UUID = UUID()
        var task: String = ""
        var status: TaskStatus = .inProgress
        var duration: Double = 0           // 소요 시간 (시간 단위)

        enum TaskStatus: String, Codable, CaseIterable {
            case todo = "todo"
            case inProgress = "in_progress"
            case done = "done"
        }
    }
}

// 회의록 템플릿
struct MeetingMinutesTemplateData: Codable {
    var meetingDate: Date = Date()
    var location: String = ""             // 회의 장소
    var participants: [String] = []       // 참석자 목록
    var agenda: String = ""              // 회의 안건
    var discussionPoints: [String] = []  // 논의 사항
    var decisions: [String] = []         // 결정 사항
    var actionItems: [ActionItem] = []   // 액션 아이템

    struct ActionItem: Codable, Identifiable {
        var id: UUID = UUID()
        var task: String = ""
        var assignee: String = ""
        var dueDate: Date = Date()
        var isCompleted: Bool = false
    }
}

// 전시회 템플릿
struct ExhibitionTemplateData: Codable {
    var exhibitionName: String = ""      // 전시회명
    var exhibitionDate: Date = Date()    // 방문 날짜
    var venue: String = ""              // 장소
    var organizer: String = ""          // 주최사
    var visitedBooths: [VisitedBooth] = [] // 방문 부스 목록
    var overallImpression: String = ""  // 전반적인 인상
    var followUpItems: [String] = []    // 후속 조치

    struct VisitedBooth: Codable, Identifiable {
        var id: UUID = UUID()
        var boothNumber: String = ""
        var companyName: String = ""
        var contactPerson: String = ""
        var productsServices: String = ""
        var interestLevel: Int = 3       // 1~5 관심도
        var notes: String = ""
    }
}
```

---

## 🔍 OCR 서비스 상세 구현

```swift
// Services/OCRService.swift
import Vision
import VisionKit
import UIKit

class OCRService {

    // 이미지에서 텍스트 인식 (한국어, 영어, 중국어 동시 지원)
    func recognizeText(
        from image: UIImage,
        completion: @escaping (Result<[String], Error>) -> Void
    ) {
        guard let cgImage = image.cgImage else {
            completion(.failure(OCRError.invalidImage))
            return
        }

        let request = VNRecognizeTextRequest { request, error in
            if let error = error {
                completion(.failure(error))
                return
            }

            let observations = request.results as? [VNRecognizedTextObservation] ?? []
            let recognizedStrings = observations.compactMap {
                $0.topCandidates(1).first?.string
            }
            completion(.success(recognizedStrings))
        }

        // 한국어, 영어, 중국어 간체/번체 모두 인식
        request.recognitionLanguages = ["ko-KR", "en-US", "zh-Hans", "zh-Hant"]
        request.recognitionLevel = .accurate
        request.usesLanguageCorrection = true

        let handler = VNImageRequestHandler(cgImage: cgImage, options: [:])
        DispatchQueue.global(qos: .userInitiated).async {
            do {
                try handler.perform([request])
            } catch {
                completion(.failure(error))
            }
        }
    }

    enum OCRError: LocalizedError {
        case invalidImage
        var errorDescription: String? {
            switch self {
            case .invalidImage: return "유효하지 않은 이미지입니다."
            }
        }
    }
}
```

```swift
// Services/BusinessCardParser.swift
import Foundation
import NaturalLanguage

class BusinessCardParser {

    // OCR 인식 텍스트 배열 → BusinessCard 모델로 파싱
    func parse(lines: [String], language: String = "ko") -> BusinessCard {
        let card = BusinessCard()
        card.scannedLanguage = language

        var remainingLines = lines

        // 이메일 패턴 추출
        let emailPattern = #"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}"#
        if let emailLine = extractPattern(emailPattern, from: &remainingLines) {
            card.email = emailLine
        }

        // 전화번호 패턴 추출 (한국 번호 형식 포함)
        let phonePattern = #"(\+82[-\s]?|0)\d{1,2}[-\s]?\d{3,4}[-\s]?\d{4}"#
        let phones = extractAllPatterns(phonePattern, from: &remainingLines)
        if phones.count >= 1 { card.phone = phones[0] }
        if phones.count >= 2 { card.officePhone = phones[1] }

        // FAX 패턴 추출
        let faxPattern = #"(?i)(fax|팩스)[\s:：]?\s*(\+82[-\s]?|0)\d{1,2}[-\s]?\d{3,4}[-\s]?\d{4}"#
        if let faxLine = extractPattern(faxPattern, from: &remainingLines) {
            card.fax = faxLine
        }

        // 웹사이트 패턴 추출
        let urlPattern = #"(https?://|www\.)[^\s]+"#
        if let url = extractPattern(urlPattern, from: &remainingLines) {
            card.website = url
        }

        // 회사명, 이름, 직함은 NaturalLanguage 기반 휴리스틱으로 처리
        parsePersonalInfo(from: remainingLines, into: card, language: language)

        return card
    }

    // 정규식으로 패턴 추출 및 remainingLines에서 제거
    private func extractPattern(_ pattern: String, from lines: inout [String]) -> String? {
        guard let regex = try? NSRegularExpression(pattern: pattern) else { return nil }
        for (index, line) in lines.enumerated() {
            let range = NSRange(line.startIndex..., in: line)
            if let match = regex.firstMatch(in: line, range: range),
               let swiftRange = Range(match.range, in: line) {
                let result = String(line[swiftRange])
                lines.remove(at: index)
                return result
            }
        }
        return nil
    }

    private func extractAllPatterns(_ pattern: String, from lines: inout [String]) -> [String] {
        var results: [String] = []
        guard let regex = try? NSRegularExpression(pattern: pattern) else { return results }
        var indicesToRemove: [Int] = []
        for (index, line) in lines.enumerated() {
            let range = NSRange(line.startIndex..., in: line)
            if let match = regex.firstMatch(in: line, range: range),
               let swiftRange = Range(match.range, in: line) {
                results.append(String(line[swiftRange]))
                indicesToRemove.append(index)
            }
        }
        for i in indicesToRemove.reversed() { lines.remove(at: i) }
        return results
    }

    private func parsePersonalInfo(from lines: [String], into card: BusinessCard, language: String) {
        // 언어별 직함 키워드 사전
        let titleKeywords_ko = ["대표", "이사", "부장", "과장", "차장", "팀장", "대리", "주임", "사원", "사장", "회장", "본부장", "실장", "선임", "책임", "수석"]
        let titleKeywords_en = ["CEO", "Director", "Manager", "President", "VP", "Senior", "Lead", "Engineer", "Consultant", "Analyst"]
        let titleKeywords_zh = ["总经理", "经理", "主任", "部长", "总监", "董事长"]

        var titleKeywords = titleKeywords_en
        if language == "ko" { titleKeywords += titleKeywords_ko }
        if language == "zh" { titleKeywords += titleKeywords_zh }

        for line in lines {
            let trimmed = line.trimmingCharacters(in: .whitespaces)
            guard !trimmed.isEmpty else { continue }

            // 직함이 포함된 줄
            if card.jobTitle.isEmpty,
               titleKeywords.contains(where: { trimmed.contains($0) }) {
                card.jobTitle = trimmed
                continue
            }

            // 이름 추정: 2~5자, 한글 또는 영문
            if card.name.isEmpty {
                let koreanNamePattern = #"^[가-힣]{2,5}$"#
                let englishNamePattern = #"^[A-Za-z]+ [A-Za-z]+"#
                if trimmed.range(of: koreanNamePattern, options: .regularExpression) != nil ||
                   trimmed.range(of: englishNamePattern, options: .regularExpression) != nil {
                    card.name = trimmed
                    continue
                }
            }

            // 회사명: 주식회사, Inc., Co., Ltd. 등 포함
            if card.company.isEmpty {
                let companyKeywords = ["주식회사", "(주)", "Inc.", "Co.", "Ltd.", "Corp.", "有限公司", "株式会社"]
                if companyKeywords.contains(where: { trimmed.contains($0) }) {
                    card.company = trimmed
                    continue
                }
            }

            // 부서: 팀, 본부, 사업부 등
            if card.department.isEmpty {
                let deptKeywords = ["팀", "본부", "사업부", "부문", "센터", "Division", "Department", "Dept"]
                if deptKeywords.contains(where: { trimmed.contains($0) }) {
                    card.department = trimmed
                    continue
                }
            }

            // 나머지는 주소로 처리
            if card.address.isEmpty && trimmed.count > 10 {
                card.address = trimmed
            }
        }
    }
}
```

---

## 📊 Excel 내보내기 서비스

```swift
// Services/ExcelExportService.swift
import Foundation

class ExcelExportService {

    // 명함 데이터 → CSV 생성 (Excel에서 열 수 있는 UTF-8 BOM CSV)
    func exportBusinessCards(
        _ cards: [BusinessCard],
        fileName: String = "businessCards"
    ) throws -> URL {

        let headers = [
            "이름(Name)", "회사(Company)", "부서(Department)",
            "직함(Title)", "이메일(Email)", "휴대폰(Mobile)",
            "사무실(Office)", "FAX", "주소(Address)",
            "웹사이트(Website)", "메모(Memo)", "스캔언어(Language)",
            "등록일(Date)"
        ]

        var csvContent = "\u{FEFF}" // UTF-8 BOM (Excel 한글 깨짐 방지)
        csvContent += headers.map { "\"\($0)\"" }.joined(separator: ",") + "\n"

        let dateFormatter = DateFormatter()
        dateFormatter.dateFormat = "yyyy-MM-dd HH:mm"

        for card in cards {
            let row = [
                card.name, card.company, card.department,
                card.jobTitle, card.email, card.phone,
                card.officePhone, card.fax, card.address,
                card.website, card.memo, card.scannedLanguage,
                dateFormatter.string(from: card.createdAt)
            ].map { "\"\($0.replacingOccurrences(of: "\"", with: "\"\""))\"" }

            csvContent += row.joined(separator: ",") + "\n"
        }

        // 임시 디렉토리에 파일 저장
        let tempURL = FileManager.default.temporaryDirectory
            .appendingPathComponent("\(fileName)_\(Date().timeIntervalSince1970).csv")

        try csvContent.write(to: tempURL, atomically: true, encoding: .utf8)
        return tempURL
    }

    // 노트 데이터 → CSV 생성
    func exportNotes(
        _ notes: [Note],
        category: NoteCategory? = nil,
        fileName: String = "notes"
    ) throws -> URL {

        let filteredNotes = category != nil
            ? notes.filter { $0.category == category }
            : notes

        let headers = [
            "제목(Title)", "카테고리(Category)", "내용(Content)",
            "태그(Tags)", "즐겨찾기(Favorite)", "생성일(Created)", "수정일(Updated)"
        ]

        var csvContent = "\u{FEFF}"
        csvContent += headers.map { "\"\($0)\"" }.joined(separator: ",") + "\n"

        let dateFormatter = DateFormatter()
        dateFormatter.dateFormat = "yyyy-MM-dd HH:mm"

        for note in filteredNotes {
            let row = [
                note.title,
                note.category.localizedName,
                note.content,
                note.tags.joined(separator: ";"),
                note.isFavorite ? "★" : "",
                dateFormatter.string(from: note.createdAt),
                dateFormatter.string(from: note.updatedAt)
            ].map { "\"\($0.replacingOccurrences(of: "\"", with: "\"\""))\"" }

            csvContent += row.joined(separator: ",") + "\n"
        }

        let tempURL = FileManager.default.temporaryDirectory
            .appendingPathComponent("\(fileName)_\(Date().timeIntervalSince1970).csv")

        try csvContent.write(to: tempURL, atomically: true, encoding: .utf8)
        return tempURL
    }
}
```

---

## 🖥️ 주요 뷰 구현 가이드

### 1. 루트 레이아웃 (iPad 사이드바 + iPhone 탭바)

```swift
// ContentView.swift
import SwiftUI

struct ContentView: View {
    @Environment(\.horizontalSizeClass) var horizontalSizeClass

    var body: some View {
        if horizontalSizeClass == .regular {
            // iPad: 3단 NavigationSplitView
            iPadLayout()
        } else {
            // iPhone: TabView
            iPhoneLayout()
        }
    }
}

// iPad 레이아웃: 사이드바(카테고리) → 목록 → 상세
private func iPadLayout() -> some View {
    NavigationSplitView {
        SidebarView()
    } content: {
        NoteListView()
    } detail: {
        EmptyStateView()
    }
}

// iPhone 레이아웃: 하단 탭바
private func iPhoneLayout() -> some View {
    TabView {
        NoteListView()
            .tabItem {
                Label("notes", systemImage: "note.text")
            }
        BusinessCardListView()
            .tabItem {
                Label("businessCards", systemImage: "person.crop.rectangle")
            }
        SettingsView()
            .tabItem {
                Label("settings", systemImage: "gearshape")
            }
    }
}
```

### 2. 명함 스캔 흐름

```swift
// Views/BusinessCard/BusinessCardScanView.swift
import SwiftUI
import VisionKit

struct BusinessCardScanView: UIViewControllerRepresentable {
    @Binding var scannedImages: [UIImage]
    @Environment(\.dismiss) var dismiss

    func makeUIViewController(context: Context) -> VNDocumentCameraViewController {
        let vc = VNDocumentCameraViewController()
        vc.delegate = context.coordinator
        return vc
    }

    func updateUIViewController(_ uiViewController: VNDocumentCameraViewController, context: Context) {}

    func makeCoordinator() -> Coordinator { Coordinator(self) }

    class Coordinator: NSObject, VNDocumentCameraViewControllerDelegate {
        let parent: BusinessCardScanView

        init(_ parent: BusinessCardScanView) { self.parent = parent }

        func documentCameraViewController(
            _ controller: VNDocumentCameraViewController,
            didFinishWith scan: VNDocumentCameraScan
        ) {
            parent.scannedImages = (0..<scan.pageCount).map { scan.imageOfPage(at: $0) }
            parent.dismiss()
        }

        func documentCameraViewControllerDidCancel(_ controller: VNDocumentCameraViewController) {
            parent.dismiss()
        }

        func documentCameraViewController(
            _ controller: VNDocumentCameraViewController,
            didFailWithError error: Error
        ) {
            parent.dismiss()
        }
    }
}
```

### 3. 명함 스캔 결과 확인 뷰

```swift
// Views/BusinessCard/BusinessCardResultView.swift
// 스캔된 명함 이미지와 파싱된 필드를 나란히 표시
// 사용자가 각 필드를 직접 수정 가능
// "노트에 추가" 버튼으로 현재 편집 중인 노트에 명함 정보 연결
// "완료" 버튼으로 명함만 독립 저장
```

### 4. 내보내기 시트

```swift
// Views/Export/ExportView.swift
// - 내보내기 대상 선택: 명함만 / 노트만 / 전체
// - 카테고리 필터 (전체 / 업무일지 / 회의록 / 전시회)
// - 날짜 범위 필터 (시작일 ~ 종료일)
// - 형식: CSV (Excel 호환)
// - 내보내기 실행 → ShareSheet (UIActivityViewController)로 공유
```

---

## 🌐 다국어 (Localizable.xcstrings)

아래 키를 최소한으로 한국어(기본)/영어 두 가지로 작성:

```json
{
  "sourceLanguage": "ko",
  "strings": {
    "category.workLog":         { "ko": "업무일지",   "en": "Work Log" },
    "category.meetingMinutes":  { "ko": "회의록",     "en": "Meeting Minutes" },
    "category.exhibition":      { "ko": "전시회",     "en": "Exhibition" },
    "category.custom":          { "ko": "기타",       "en": "Others" },
    "nav.notes":                { "ko": "노트",       "en": "Notes" },
    "nav.businessCards":        { "ko": "명함",       "en": "Business Cards" },
    "nav.settings":             { "ko": "설정",       "en": "Settings" },
    "action.newNote":           { "ko": "새 노트",    "en": "New Note" },
    "action.scanCard":          { "ko": "명함 스캔",  "en": "Scan Card" },
    "action.export":            { "ko": "내보내기",   "en": "Export" },
    "action.save":              { "ko": "저장",       "en": "Save" },
    "action.cancel":            { "ko": "취소",       "en": "Cancel" },
    "action.delete":            { "ko": "삭제",       "en": "Delete" },
    "export.title":             { "ko": "데이터 내보내기", "en": "Export Data" },
    "export.businessCards":     { "ko": "명함 데이터", "en": "Business Cards" },
    "export.notes":             { "ko": "노트 데이터", "en": "Notes" },
    "export.all":               { "ko": "전체",       "en": "All" },
    "businessCard.name":        { "ko": "이름",       "en": "Name" },
    "businessCard.company":     { "ko": "회사",       "en": "Company" },
    "businessCard.department":  { "ko": "부서",       "en": "Department" },
    "businessCard.jobTitle":    { "ko": "직함",       "en": "Title" },
    "businessCard.email":       { "ko": "이메일",     "en": "Email" },
    "businessCard.phone":       { "ko": "휴대폰",     "en": "Mobile" },
    "businessCard.officePhone": { "ko": "사무실",     "en": "Office" },
    "businessCard.address":     { "ko": "주소",       "en": "Address" },
    "businessCard.website":     { "ko": "웹사이트",   "en": "Website" },
    "businessCard.memo":        { "ko": "메모",       "en": "Memo" },
    "template.workLog.items":   { "ko": "업무 항목",  "en": "Work Items" },
    "template.workLog.achievements": { "ko": "주요 성과", "en": "Achievements" },
    "template.workLog.issues":  { "ko": "이슈/문제점","en": "Issues" },
    "template.workLog.nextTodos": { "ko": "다음 할 일", "en": "Next To-Do" },
    "template.meeting.location":{ "ko": "회의 장소",  "en": "Location" },
    "template.meeting.participants": { "ko": "참석자", "en": "Participants" },
    "template.meeting.agenda":  { "ko": "안건",       "en": "Agenda" },
    "template.meeting.decisions": { "ko": "결정 사항","en": "Decisions" },
    "template.meeting.actions": { "ko": "액션 아이템","en": "Action Items" },
    "template.exhibition.name": { "ko": "전시회명",   "en": "Exhibition Name" },
    "template.exhibition.venue":{ "ko": "장소",       "en": "Venue" },
    "template.exhibition.booths": { "ko": "방문 부스","en": "Visited Booths" }
  }
}
```

---

## ⚙️ 앱 설정 및 권한

### Info.plist 필수 키
```xml
<!-- 카메라 권한 (명함 스캔) -->
<key>NSCameraUsageDescription</key>
<string>명함을 스캔하여 연락처 정보를 자동으로 입력합니다.</string>
<!-- 사진 라이브러리 권한 (첨부파일) -->
<key>NSPhotoLibraryUsageDescription</key>
<string>노트에 사진을 첨부할 수 있습니다.</string>
```

### Capabilities (Xcode Signing & Capabilities)
- iCloud (CloudKit) — SwiftData 동기화
- Push Notifications — 알림 (선택)

---

## 🎨 디자인 가이드라인

- **디자인 시스템**: Apple Human Interface Guidelines 100% 준수
- **색상**: 카테고리별 Accent Color (Assets.xcassets에 Light/Dark 모두 정의)
  - 업무일지: 파란색 `#2563EB`
  - 회의록: 초록색 `#059669`
  - 전시회: 주황색 `#D97706`
  - 기타: 보라색 `#7C3AED`
- **타이포그래피**: SF Pro (시스템 기본) — Dynamic Type 완전 지원
- **다크모드**: 모든 뷰에서 지원 (`.preferredColorScheme` 고려)
- **iPad 멀티태스킹**: Split View, Slide Over 지원
- **접근성**: VoiceOver label, .accessibilityLabel 추가

---

## 📋 개발 단계별 구현 순서

### Phase 1 — 기반 구조 (1~2일)
1. Xcode 프로젝트 생성 (SwiftUI, SwiftData, CloudKit 활성화)
2. SwiftData 모델 정의 (`Note`, `BusinessCard`)
3. 기본 앱 구조 (`ContentView`, iPad/iPhone 레이아웃 분기)
4. Localizable.xcstrings 설정 (한국어/영어)

### Phase 2 — 노트 CRUD (2~3일)
5. 카테고리 목록/선택 뷰
6. 노트 목록 뷰 (검색, 필터, 정렬)
7. 노트 편집기 (자유 텍스트 + 카테고리별 구조화 템플릿)
8. 즐겨찾기, 태그 기능

### Phase 3 — 명함 스캔 (2~3일)
9. VNDocumentCameraViewController 연동
10. OCRService 구현 (한국어/영어/중국어)
11. BusinessCardParser 구현 (정규식 + 휴리스틱)
12. 스캔 결과 확인/수정 뷰
13. 명함 → 노트 연결 기능

### Phase 4 — 내보내기 (1일)
14. ExcelExportService (CSV) 구현
15. 내보내기 필터 UI
16. UIActivityViewController 연동

### Phase 5 — 마감 작업 (1~2일)
17. iCloud 동기화 테스트
18. 다크모드 / Dynamic Type 검증
19. iPad 멀티태스킹 검증
20. 앱 아이콘 / 스플래시 화면

---

## ✅ 구현 시 주의사항

1. **SwiftData `@Model`에는 enum을 직접 저장 불가** → `rawValue: String`으로 저장 후 computed property로 변환
2. **VNDocumentCameraViewController는 시뮬레이터에서 동작 안 함** → 실기기 테스트 필수, 시뮬레이터용 Mock 데이터 준비
3. **OCR 정확도** → `recognitionLevel = .accurate` 설정, 조명/각도에 따라 결과 편차 발생하므로 수동 수정 UI 반드시 제공
4. **CSV UTF-8 BOM** → Excel에서 한글 깨짐 방지를 위해 파일 앞에 `\u{FEFF}` 반드시 추가
5. **SwiftData + CloudKit** → `@Model` 프로퍼티에 Optional 또는 기본값 필수 (CloudKit 스키마 제약)
6. **명함 이미지 저장** → SwiftData에 Data 직접 저장 지양, FileManager 경로 저장 후 참조
7. **iPad NavigationSplitView** → `columnVisibility` 상태 관리로 사이드바 자동 접힘 처리

---

## 🚀 Claude Code 실행 명령어

```bash
# 1. 프로젝트 생성 (Xcode CLI)
xcodebuild -create-xcodeproj BizNote

# 2. 의존성 추가 (Package.swift 또는 Xcode UI)
# File > Add Package Dependencies > XLSXF URL 입력

# 3. 시뮬레이터 빌드 확인
xcodebuild -scheme BizNote -destination 'platform=iOS Simulator,name=iPhone 16 Pro' build

# 4. iPad 시뮬레이터 빌드 확인
xcodebuild -scheme BizNote -destination 'platform=iOS Simulator,name=iPad Pro 13-inch (M4)' build
```

---

*이 프롬프트를 Claude Code에 전달하여 BizNote 앱을 단계적으로 구현하세요.*
*각 Phase가 완료될 때마다 빌드 오류 없이 컴파일되는지 확인 후 다음 단계로 진행하세요.*
