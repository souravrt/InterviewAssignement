# Assignment
# AAOS UI Customization: Separate Quick Favorites and Notification Center

## Architecture Documentation

### 1. Introduction and Goals

#### 1.1 Requirements Overview

The current Android Automotive OS (AAOS) implementation combines Notifications and Quick Favorites in a single overlay accessible through downward swipe gestures. This architecture document outlines the design for separating these components into distinct overlays with independent gesture controls:

- **Swipe down (top to bottom)**: Opens Notification Center overlay
- **Swipe up (bottom to top)**: Opens Quick Favorites overlay

#### 1.2 Quality Goals

| Priority | Quality Goal | Description |
|----------|-------------|-------------|
| 1 | **Performance** | Zero latency gesture response, minimal memory footprint |
| 2 | **Usability** | Intuitive gesture-based access, accessibility compliance |
| 3 | **Maintainability** | Modular architecture, clean separation of concerns |
| 4 | **Compatibility** | Multi-display support (Center, Passenger, Rear Seats) |
| 5 | **Localization** | Support for diverse languages and regions |

#### 1.3 Stakeholders

| Role | Expectation |
|------|-------------|
| **End Users** | Intuitive, fast access to notifications and quick settings |
| **OEM Integrators** | Customizable, brandable interface components |
| **System Architects** | Maintainable, scalable architecture |
| **QA Engineers** | Testable, reliable gesture handling |

---

### 2. Architecture Constraints

#### 2.1 Technical Constraints
- **Platform**: Android Automotive OS 14.0.0 LTS7
- **Framework**: CarSystemUI architecture
- **Memory**: Limited RAM on automotive hardware
- **Performance**: <100ms gesture response time
- **Multi-display**: Support for up to 4 displays simultaneously

#### 2.2 Organizational Constraints
- **Code Base**: Must maintain compatibility with existing CarServices
- **Standards**: Follow Android Automotive design guidelines
- **Development**: Use existing SystemUIOverlayWindow management system

#### 2.3 Regulatory Constraints
- **Safety**: Comply with automotive safety standards (ISO 26262)
- **Accessibility**: WCAG 2.1 AA compliance for in-vehicle use
- **Localization**: Support RTL languages and regional preferences

---

### 3. System Scope and Context

#### 3.1 Business Context

```
┌─────────────────┐    ┌──────────────────────┐    ┌─────────────────┐
│   Driver/User   │◄──►│  AAOS UI System      │◄──►│  Vehicle ECUs   │
│                 │    │                      │    │                 │
│ • Gesture Input │    │ • Notification Center│    │ • Climate       │
│ • Visual Output │    │ • Quick Favorites    │    │ • Media         │
│ • Voice Commands│    │ • Multi-Display Mgmt │    │ • Navigation    │
└─────────────────┘    └──────────────────────┘    └─────────────────┘
          │                        │                        │
          │                        │                        │
          ▼                        ▼                        ▼
┌─────────────────┐    ┌──────────────────────┐    ┌─────────────────┐
│  Input Devices  │    │   CarSystemUI        │    │   Car Services  │
│                 │    │                      │    │                 │
│ • Touch Screen  │    │ • Overlay Windows    │    │ • Property Mgmt │
│ • Physical Keys │    │ • Gesture Detection  │    │ • Power Mgmt    │
│ • Voice Input   │    │ • Animation Engine   │    │ • Sensor Data   │
└─────────────────┘    └──────────────────────┘    └─────────────────┘
```

#### 3.2 Technical Context

The system operates within the AAOS framework, interfacing with:

- **SystemUIOverlayWindow**: Primary overlay management system
- **NotificationManager**: Android notification framework
- **InputManager**: Gesture and touch event processing
- **DisplayManager**: Multi-display coordination
- **CarPropertyManager**: Vehicle state integration

---

### 4. Solution Strategy

#### 4.1 Architectural Approach

**Separation of Concerns**: Create two independent overlay systems:
1. **NotificationCenterOverlay**: Manages notification display and interactions
2. **QuickFavoritesOverlay**: Handles quick settings and shortcuts

**Gesture-Driven Architecture**: Implement directional gesture recognition:
- Vertical swipe detection with velocity thresholds
- Independent gesture handlers for each overlay
- Conflict resolution for simultaneous gestures

**Multi-Display Strategy**: 
- Per-display overlay instance management
- Synchronized state across displays
- Display-specific gesture calibration

#### 4.2 Technology Decisions

| Component | Technology Choice | Rationale |
|-----------|------------------|-----------|
| **Overlay Framework** | SystemUIOverlayWindow | Existing AAOS framework, Z-order management |
| **Gesture Detection** | Custom GestureDetector | Performance, automotive-specific requirements |
| **Animation Engine** | Android Property Animation | Smooth 60fps transitions |
| **State Management** | Mediator Pattern | Clean separation, testability |
| **Multi-Display** | DisplayContent per instance | Isolated display management |

---

### 5. Building Block View

#### 5.1 Level 1: System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    CarSystemUI Application                       │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐              ┌─────────────────────────┐   │
│  │ Gesture Handler │◄────────────►│  Overlay Manager        │   │
│  │                 │              │                         │   │
│  │ • Swipe Up      │              │ • Z-Order Management    │   │
│  │ • Swipe Down    │              │ • Animation Control     │   │
│  │ • Velocity Calc │              │ • State Coordination    │   │
│  └─────────────────┘              └─────────────────────────┘   │
│           │                                    │                │
│           ▼                                    ▼                │
│  ┌─────────────────┐              ┌─────────────────────────┐   │
│  │Notification     │              │ Quick Favorites         │   │
│  │Center Overlay   │              │ Overlay                 │   │
│  │                 │              │                         │   │
│  │ • HUN Display   │              │ • Settings Tiles        │   │
│  │ • List View     │              │ • Custom Actions        │   │
│  │ • Interactions  │              │ • Vehicle Controls      │   │
│  └─────────────────┘              └─────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│                    Android Framework Layer                       │
│  NotificationManager | InputManager | DisplayManager | CarAPI   │
└─────────────────────────────────────────────────────────────────┘
```

#### 5.2 Level 2: Detailed Components

##### 5.2.1 Gesture Handler Module

```java
public class VehicleGestureController {
    private static final int SWIPE_THRESHOLD = 100;
    private static final int VELOCITY_THRESHOLD = 100;
    
    public enum SwipeDirection {
        UP_TO_DOWN,    // Notification Center
        DOWN_TO_UP,    // Quick Favorites  
        LEFT_TO_RIGHT,
        RIGHT_TO_LEFT
    }
    
    // Gesture detection with automotive-specific thresholds
    public void onGestureDetected(SwipeDirection direction, float velocity);
}
```

##### 5.2.2 Overlay Window Architecture

```java
// Notification Center Overlay
public class NotificationCenterOverlayViewController extends OverlayViewController {
    private RecyclerView mNotificationList;
    private AnimationController mAnimationController;
    
    @Override
    protected void showInternal() {
        // Slide down animation from top
        mAnimationController.slideFromTop(ANIMATION_DURATION);
    }
}

// Quick Favorites Overlay  
public class QuickFavoritesOverlayViewController extends OverlayViewController {
    private GridView mQuickTileGrid;
    private VehicleControlPanel mVehicleControls;
    
    @Override
    protected void showInternal() {
        // Slide up animation from bottom
        mAnimationController.slideFromBottom(ANIMATION_DURATION);
    }
}
```

---

### 6. Runtime View

#### 6.1 Gesture Processing Flow

```
User Gesture Input
        │
        ▼
┌─────────────────┐
│ Touch Event     │
│ Processing      │
└─────────┬───────┘
          │
          ▼
┌─────────────────┐     ┌─────────────────┐
│ Gesture         │────►│ Direction &     │
│ Recognition     │     │ Velocity Calc   │
└─────────────────┘     └─────────┬───────┘
                                  │
                ┌─────────────────┼─────────────────┐
                │                 │                 │
                ▼                 ▼                 ▼
        ┌───────────────┐ ┌──────────────┐ ┌──────────────┐
        │ Swipe Up      │ │ Swipe Down   │ │ Other Gesture│
        │ (Quick Fav)   │ │ (Notification│ │ (Ignored)    │
        └───────┬───────┘ └──────┬───────┘ └──────────────┘
                │                │
                ▼                ▼
    ┌─────────────────┐ ┌─────────────────┐
    │ Show Quick      │ │ Show Notification│
    │ Favorites       │ │ Center          │
    │ Overlay         │ │ Overlay         │
    └─────────────────┘ └─────────────────┘
```

#### 6.2 Multi-Display Coordination

```
Primary Display          Secondary Display         Rear Display
      │                        │                       │
      ▼                        ▼                       ▼
┌─────────────┐        ┌─────────────┐         ┌─────────────┐
│Display      │        │Display      │         │Display      │
│Manager A    │◄──────►│Manager B    │◄───────►│Manager C    │
└─────────────┘        └─────────────┘         └─────────────┘
      │                        │                       │
      ▼                        ▼                       ▼
┌─────────────┐        ┌─────────────┐         ┌─────────────┐
│Overlay      │        │Overlay      │         │Overlay      │
│Instance A   │        │Instance B   │         │Instance C   │
└─────────────┘        └─────────────┘         └─────────────┘
      │                        │                       │
      └────────────────────────┼───────────────────────┘
                               │
                               ▼
                    ┌─────────────────┐
                    │ State           │
                    │ Synchronizer    │
                    └─────────────────┘
```

---

### 7. Deployment View

#### 7.1 Physical Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Vehicle Hardware                          │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │   Center    │  │  Passenger  │  │    Rear Seats       │ │
│  │   Display   │  │   Display   │  │    Display          │ │
│  │             │  │             │  │                     │ │
│  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────────────┐ │ │
│  │ │ Overlay │ │  │ │ Overlay │ │  │ │ Overlay         │ │ │
│  │ │ Window  │ │  │ │ Window  │ │  │ │ Window          │ │ │
│  │ │ Stack   │ │  │ │ Stack   │ │  │ │ Stack           │ │ │
│  │ └─────────┘ │  │ └─────────┘ │  │ └─────────────────┘ │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                 Android Automotive OS                        │
│                                                             │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │              CarSystemUI Process                        │ │
│  │                                                         │ │
│  │ ┌──────────────┐ ┌──────────────┐ ┌─────────────────┐  │ │
│  │ │ Notification │ │ Quick        │ │ Gesture         │  │ │
│  │ │ Center       │ │ Favorites    │ │ Handler         │  │ │
│  │ │ Service      │ │ Service      │ │ Service         │  │ │
│  │ └──────────────┘ └──────────────┘ └─────────────────┘  │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

#### 7.2 Software Deployment

| Component | Process | Location | Dependencies |
|-----------|---------|----------|-------------|
| **GestureHandler** | CarSystemUI | /system/priv-app/CarSystemUI | InputManager, MotionEvent |
| **NotificationOverlay** | CarSystemUI | /system/priv-app/CarSystemUI | NotificationManager, SystemUI |
| **QuickFavoritesOverlay** | CarSystemUI | /system/priv-app/CarSystemUI | CarPropertyManager, Settings |
| **OverlayManager** | CarSystemUI | /system/priv-app/CarSystemUI | WindowManager, DisplayManager |

---

### 8. Cross-cutting Concepts

#### 8.1 Performance Optimization

**Memory Management**:
- Lazy initialization of overlay components
- Bitmap recycling for notification images
- View holder pattern for list recycling

**Gesture Performance**:
- Hardware-accelerated touch processing
- Predictive gesture recognition
- Frame rate optimization (60fps target)

#### 8.2 Security Considerations

**Permission Management**:
- Runtime permission checking for vehicle properties
- Secure overlay window token validation
- Multi-user isolation for personal data

**Data Protection**:
- Notification content sanitization
- Encrypted storage for user preferences
- Secure IPC between overlay components

#### 8.3 Accessibility Implementation

**Input Accessibility**:
- Voice command integration for overlay control
- Large touch targets (minimum 44dp)
- High contrast theme support

**Output Accessibility**:
- Screen reader compatibility
- Haptic feedback for gesture confirmation
- Audio cues for overlay state changes

---

### 9. Architecture Decisions

#### 9.1 ADR-001: Separate Overlay Architecture

**Date**: N/A  
**Status**: Draft

**Context**: Current unified overlay creates gesture conflicts and limits customization options.

**Decision**: Implement separate SystemUIOverlayWindow instances for Notification Center and Quick Favorites.

**Consequences**:
- ✅ Independent gesture handling
- ✅ Separate customization paths  
- ✅ Reduced complexity per overlay
- ❌ Increased memory footprint
- ❌ Additional state synchronization

#### 9.2 ADR-002: Custom Gesture Detection

**Date**: N/A  
**Status**: Draft

**Context**: Standard Android gesture detection not optimized for automotive use cases.

**Decision**: Implement custom gesture recognition with automotive-specific thresholds and safety considerations.

**Consequences**:
- ✅ Optimized for in-vehicle use
- ✅ Configurable sensitivity per display
- ✅ Safety-compliant response times
- ❌ Additional testing complexity
- ❌ Custom maintenance overhead

#### 9.3 ADR-003: Multi-Display Instance Management

**Date**: N/A  
**Status**: Draft

**Context**: AAOS supports multiple displays with independent user interactions.

**Decision**: Create separate overlay instances per display with centralized state coordination.

**Consequences**:
- ✅ Independent display behavior
- ✅ Scalable multi-display support
- ✅ Display-specific customization
- ❌ Increased resource usage
- ❌ Complex state synchronization

---

### 10. Quality Requirements

#### 10.1 Performance Requirements

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Gesture Response** | <100ms | Touch to overlay animation start |
| **Memory Usage** | <50MB total | Both overlays loaded |
| **Frame Rate** | 60fps | During animations |
| **CPU Usage** | <5% continuous | Background gesture monitoring |

#### 10.2 Usability Requirements

| Requirement | Implementation | Validation |
|-------------|----------------|------------|
| **Intuitive Gestures** | Standard swipe directions | User testing |
| **Visual Feedback** | Immediate animation response | Automated testing |
| **Accessibility** | Voice commands, high contrast | Compliance testing |
| **Localization** | RTL support, multi-language | Regional testing |

---

### 11. Risks and Technical Debt

#### 11.1 Identified Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| **Gesture Conflicts** | Medium | High | Comprehensive gesture detection logic |
| **Performance Degradation** | Low | High | Continuous performance monitoring |
| **Multi-Display Sync Issues** | Medium | Medium | Robust state management architecture |
| **Memory Leaks** | Low | Medium | Strict lifecycle management |

#### 11.2 Technical Debt

- **Legacy Integration**: Current unified overlay code requires gradual migration
- **Test Coverage**: Comprehensive gesture testing framework needed
- **Documentation**: Multi-display configuration documentation incomplete
- **Performance Profiling**: Automotive-specific performance benchmarks required

---

### 12. Glossary

| Term | Definition |
|------|------------|
| **AAOS** | Android Automotive OS - Google's automotive platform |
| **HUN** | Heads-Up Notification - Priority notifications overlaying content |
| **SystemUIOverlayWindow** | AAOS framework for system-level overlay management |
| **VHAL** | Vehicle Hardware Abstraction Layer |
| **CarSystemUI** | Automotive-specific SystemUI implementation |
| **Z-Order** | Window stacking order for display priority |
| **RTL** | Right-to-Left text direction for certain languages |

---
