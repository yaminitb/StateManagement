1.1 `@Published` + `ObservableObject` as the single source of truth

@MainActor
final class SessionManager: ObservableObject {
    static let shared = SessionManager()
 
    @Published var showInactivityWarning: Bool = false
    @Published var showLogoutConfirmation: Bool = false
    @Published var remainingWarningSeconds: Int = 20
 
    // Views observing SessionManager re-render automatically
    // whenever any of the above changes — no manual notification needed.
}

1.2 Persisted state via a computed property (survives app restarts)

private var lastActivityDate: Date {
    get { UserDefaults.standard.object(forKey: lastActivityKey) as? Date ?? Date() }
    set { UserDefaults.standard.set(newValue, forKey: lastActivityKey) }
}

Why it's a good pattern: reads/writes look like plain property access everywhere else in the class, but are transparently backed by UserDefaults — state survives a cold launch without any extra plumbing at call sites.

1.3 Cross-feature events via `NotificationCenter`, consumed as app-wide state resets
extension NSNotification.Name {
    static let sessionExpired = NSNotification.Name("sessionExpired")
}
 
// Publisher side (SessionManager), after a failed refresh / explicit logout:
NotificationCenter.default.post(name: .sessionExpired, object: nil)
 
// Subscriber side (Router), resets navigation state app-wide:
NotificationCenter.default.addObserver(forName: .sessionExpired, object: nil, queue: .main) { [weak self] _ in
    self?.resetToRoot()
}

Why it's a good pattern: SessionManager (auth state) and Router (navigation state) don't need direct references to each other — a broadcast event keeps two independent pieces of app state in sync.

1.4 Scoped, per-screen state vs. app-wide state

Contrast the app-wide singletons above with OTPVerificationViewModel, whose @Published properties (otpCode, timerString, viewState) are scoped to a single bottom sheet and destroyed with it:
@MainActor
final class OTPVerificationViewModel: ObservableObject {
    @Published var otpCode: [String] = Array(repeating: "", count: 4)
    @Published var timerString: String = "03:00"
    @Published var viewState: ViewState = .idle
}

Takeaway: the codebase draws a clear line — singletons (SessionManager, NetworkMonitor) hold state that genuinely spans the whole app lifecycle; everything else lives in a per-screen ObservableObject created fresh by the DependencyContainer factory methods.

