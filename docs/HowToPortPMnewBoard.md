## TizenRT Power Management (PM) Porting Guide

### 1. Introduction

This guide provides instructions for porting the TizenRT Power Management (PM) subsystem to a new board or System on a Chip (SoC). The TizenRT PM framework is designed with a layered architecture:

*   **Generic PM Core (`os/pm/`)**: This layer contains the state machine logic, driver callback management, and interaction with the system's idle loop. It is largely architecture-independent and requires no modification for a new port.
*   **Board-Specific Layer (`os/arch/<chip>/<board>/`)**: This layer is responsible for implementing the low-level hardware operations required to enter and exit low-power states. This is the primary focus of the porting effort.

The porting process involves creating a set of board-specific functions that control the SoC's power management units (PMUs), clocks, and CPU idle states, and then registering these functions with the generic PM core.

### 2. Prerequisites

Before starting the port, ensure you have:

*   A working TizenRT board port for your target hardware.
*   Detailed knowledge of your SoC's power management features, including:
    *   CPU idle and sleep instructions.
    *   Clock gating and control.
    *   Power domain control.
    *   Wake-up sources (GPIOs, timers, RTC, etc.) and how to configure them.
    *   Any necessary sequences for entering and exiting sleep/standby modes.
    *   The behavior of timers and other peripherals during low-power states.

### 3. Kconfig Configuration

You need to enable and configure PM support through the Kconfig system. These settings are typically found in `os/arch/<chip>/Kconfig` and `os/board/<board_name>/Kconfig`.

#### Essential Kconfig Options:

*   `CONFIG_PM`: The master switch. Set this to `y` to enable the entire PM subsystem.
    ```kconfig
    config CONFIG_PM
        bool "Enable Power Management (PM) support"
        default n
    ```
*   `CONFIG_PM_NDOMAINS`: Defines the number of power domains. The default is 32. Adjust this based on your SoC's capabilities if needed.
    ```kconfig
    config CONFIG_PM_NDOMAINS
        int "Number of PM domains"
        default 32
        depends on CONFIG_PM
    ```

#### Optional Kconfig Options:

*   `CONFIG_PM_DVFS`: Enable support for Dynamic Voltage and Frequency Scaling. Implement this if your SoC supports changing CPU clock speeds/voltages for power savings.
    ```kconfig
    config CONFIG_PM_DVFS
        bool "Enable DVFS support"
        default n
        depends on CONFIG_PM
    ```
*   `CONFIG_PM_TIMEDWAKEUP`: Enable support for timed wake-ups from sleep. This requires a hardware timer capable of waking the CPU from the deepest sleep state.
    ```kconfig
    config CONFIG_PM_TIMEDWAKEUP
        bool "Enable Timed Wakeup"
        default n
        depends on CONFIG_PM
    ```
*   `CONFIG_PM_METRICS`: Enable PM metrics collection for debugging and profiling. This is useful during development but can be disabled for production builds.
    ```kconfig
    config CONFIG_PM_METRICS
        bool "Enable PM Metrics"
        default n
        depends on CONFIG_PM
    ```
*   `CONFIG_IDLE_CUSTOM`: If your board has highly specific idle-loop requirements not met by the standard PM logic, you can provide a custom idle loop. For most ports, this is not necessary, as the standard `pm_idle()` integration is sufficient.

### 4. Board-Specific Implementation

This is the core of the porting work. You will need to create a few C files and potentially one assembly file to implement the low-level PM functions.

#### 4.1. Implement `struct pm_sleep_ops`

This structure is the contract between your board-specific code and the generic PM core. You must define an instance of it and populate its function pointers.

**Location:** Typically in a file like `os/arch/<chip>/<board>/<board>_pm.c` or `os/arch/<chip>/<board>/<board>_idle.c`.

**Structure Definition (`os/include/tinyara/pm/pm.h`):**
```c
struct pm_sleep_ops {
    void (*sleep)(pm_wakehandler_t handler);
    void (*set_timer)(unsigned int delay_us);
};
```

**Implementation Steps:**

1.  **Create the `sleep` function:**
    *   **Purpose:** This is the main function that puts the CPU into a low-power state. It will be called by the generic PM core when a transition to `PM_SLEEP` is required.
    *   **Signature:** `void your_board_sleep(pm_wakehandler_t handler)`
    *   **`handler` Argument:** This is a pointer to the generic `pm_wakehandler` function. **Your board-specific sleep code MUST call this function upon wake-up.** It should be called from an interrupt context or immediately after the CPU resumes execution.
    *   **`handler` Arguments to Pass:**
        *   `clock_t missing_tick`: The number of system ticks that were missed during sleep. You must calculate this based on your SoC's timer behavior.
        *   `pm_wakeup_reason_code_t wakeup_src`: An enum indicating the source of the wake-up (e.g., `PM_WAKEUP_GPIO`, `PM_WAKEUP_HW_TIMER`). You must determine this by reading your SoC's wake-up status registers.
    *   **Typical `sleep` function logic:**
        1.  **Pre-sleep preparations:**
            *   Disable system tick timer and other non-wake-up interrupts.
            *   Configure wake-up sources (e.g., enable specific GPIO pins as wake-up interrupts).
            *   If `CONFIG_PM_TIMEDWAKEUP` is active, the `set_timer` function will have already been called. Your `sleep` function might need to ensure this timer is correctly armed.
        2.  **Enter low-power state:**
            *   This often involves a call to an assembly function. For example: `asm_wfi()` for Wait-For-Interrupt, or a more complex sequence for deep sleep.
            *   Example from `amebasmart_idle.c`:
                ```c
                void up_pm_board_sleep(void (*wakeuphandler)(clock_t, pm_wakeup_reason_code_t))
                {
                    // ... disable timers, mask interrupts ...
                    config_SLEEP_PROCESSING(wakeuphandler); // Calls assembly sleep
                    // ... re-enable timers, unmask interrupts ...
                }
                ```
        3.  **Post-wake-up execution:**
            *   The code here runs after the CPU wakes up.
            *   Determine the `wakeup_src` by reading SoC status registers.
            *   Calculate `missing_tick` by checking the system timer.
            *   **Crucially, call the passed-in `handler`:** `handler(missing_tick, wakeup_src);`
            *   Re-initialize any hardware that was reset or lost context during sleep (e.g., system timers).
            *   Restore interrupt masks.

2.  **Create the `set_timer` function (if `CONFIG_PM_TIMEDWAKEUP` is enabled):**
    *   **Purpose:** Program a hardware timer to wake the CPU after a specified duration.
    *   **Signature:** `void your_board_set_timer(unsigned int delay_us)`
    *   **Logic:**
        1.  Select a suitable hardware timer on your SoC that can function in low-power modes.
        2.  Initialize and configure this timer.
        3.  Set it to trigger an interrupt (or a wake-up event) after `delay_us` microseconds.
        4.  Ensure this timer's interrupt is configured as a wake-up source.
    *   **Example from `amebasmart_pmhelpers.c`:**
        ```c
        void up_set_pm_timer(unsigned int interval_us) 
        {
            if (interval_us > 0) {
                gtimer_init(&g_timer1, TIMER1); // Use specific timer TIMER1
                gtimer_start_one_shout(&g_timer1, interval_us, NULL, (uint32_t)&g_timer1);
            }
        }
        ```

3.  **Instantiate the `struct pm_sleep_ops`:**
    ```c
    // In your board's PM C file
    struct pm_sleep_ops your_board_sleep_ops = {
        .sleep = your_board_sleep,       // Pointer to your sleep function
        .set_timer = your_board_set_timer // Pointer to your set_timer function
    };
    ```

#### 4.2. Implement `struct pm_clock_ops` (Conditional)

If you enabled `CONFIG_PM_DVFS`, you must also implement this structure.

**Structure Definition (`os/include/tinyara/pm/pm.h`):**
```c
struct pm_clock_ops {
    void (*adjust_dvfs)(int div_lvl);
};
```

**Implementation Steps:**

1.  **Create the `adjust_dvfs` function:**
    *   **Purpose:** Change the CPU's clock frequency and/or voltage according to a division level.
    *   **Signature:** `void your_board_adjust_dvfs(int div_lvl)`
    *   **Logic:** This function will interact with your SoC's Clock Generation Unit (CGU) and/or Voltage Regulator Module (VRM) to set the operating point defined by `div_lvl`. The meaning of `div_lvl` (e.g., 0 for full speed, 1 for half speed, etc.) is specific to your SoC and should be documented.

2.  **Instantiate the `struct pm_clock_ops`:**
    ```c
    // In your board's PM C file
    #ifdef CONFIG_PM_DVFS
    struct pm_clock_ops your_board_clock_ops = {
        .adjust_dvfs = your_board_adjust_dvfs
    };
    #endif
    ```

#### 4.3. Assembly-Level Sleep Function (If Required)

For deep sleep states that involve significant power domain switching or CPU state loss, a portion of the sleep sequence may need to be written in assembly.

*   **Location:** Typically in a file like `os/arch/<chip>/<board>/<chip>_pm.S`.
*   **Content:** This file would contain the actual instructions to halt the CPU, switch power domains, and ensure the correct execution flow upon wake-up. The C `sleep` function would call into this assembly routine.
*   **Wake-up Handler:** The assembly code must be designed to jump to the `pm_wakehandler_t` function (passed as an argument) after the CPU has resumed execution but before returning to the C context. This often involves saving/restoring more context than a standard interrupt.

#### 4.4. Helper Functions (Recommended)

It's good practice to create helper functions for common PM tasks:

*   **Wake-up Source Configuration:** Functions to enable/disable specific wake-up pins or events (e.g., `your_board_config_gpio_wake(pin, enable)`). These will write to specific PMC registers.
*   **Wake-up Reason Checking:** A function to read the SoC's wake-up status register and return a `pm_wakeup_reason_code_t`. This is called by your `sleep` function.
*   **BSP Domain Management:** Wrappers around `pm_domain_register`, `pm_suspend`, and `pm_resume` to simplify their use by other drivers in your BSP.

### 5. Initialization

You must initialize the PM subsystem during the board's bring-up sequence.

**Location:** Typically in `os/board/<board_name>/src/<board_name>_bringup.c`.

**Steps:**

1.  **Create an initialization function:**
    ```c
    // In your board's bringup C file
    #include <tinyara/pm/pm.h>
    // Include the header where you defined your_board_sleep_ops

    void your_board_pm_initialize(void)
    {
        // Register the sleep operations with the generic PM core
        pm_initialize(&your_board_sleep_ops);

    #ifdef CONFIG_PM_DVFS
        // Register the clock operations if DVFS is enabled
        pm_clock_initialize(&your_board_clock_ops);
    #endif
    }
    ```

2.  **Call the initialization function:**
    Call `your_board_pm_initialize()` from the board's `board_bringup()` function, early in the boot process but after basic clock and memory initialization.

### 6. Integrating with the Idle Loop

The generic PM logic is driven by the system's idle loop. You need to ensure that `pm_idle()` is called when the system has no work to do.

**Location:** The architecture-specific idle function, typically `os/arch/<chip>/common/<chip>_idle.c`.

**Standard Integration (Recommended):**

Modify the `up_idle(void)` function to call the generic `pm_idle()`.

```c
#include <tinyara/pm/pm.h>

void up_idle(void)
{
#if defined(CONFIG_PM)
    // The generic PM idle function will check state, call drivers, and sleep
    pm_idle();
#else
    // Fallback for when PM is not configured
    __asm__ __volatile__("wfi");
#endif
}
```
This is the simplest and most common integration. The `pm_idle()` function contains all the logic to check `suspend_count`, call `pm_changestate`, and eventually call your board-specific `sleep` function.

### 7. Driver Adaptation

Drivers for peripherals on your board need to be made PM-aware. This involves two main aspects:

1.  **Registering for PM Callbacks:**
    *   Drivers that need to save/restore state or perform actions during power state transitions must register a `struct pm_callback_s` with the PM core using `pm_register()`.
    *   They should implement the `prepare` and `notify` callbacks to handle state changes.

2.  **Using `pm_suspend`/`pm_resume`:**
    *   Drivers that are active and cannot tolerate their power domain being turned off (or the CPU sleeping) should call `pm_suspend(domain_id)` when they start an operation and `pm_resume(domain_id)` when they finish. This increments/decrements the `suspend_count`, which `pm_checkstate()` uses to decide if the system can sleep.

### 8. Testing and Debugging

*   **Enable `CONFIG_PM_METRICS`**: This provides valuable statistics on time spent in each state and wake-up frequencies.
*   **Use `pmdbg()` and `pmvdbg()`**: These are debug logging macros for PM. Enable debug output in your Kconfig.
*   **Test State Transitions**: Manually trigger state changes if possible, or create scenarios that should lead to idle, standby, and sleep.
*   **Verify Wake-up Sources**: Ensure that all configured wake-up sources (GPIOs, timers, etc.) can successfully wake the system from sleep.
*   **Check Timer Accuracy**: After waking from sleep, verify that the system time is still accurate. The `missing_tick` calculation in your `sleep` function is critical here.

By following these steps, you can successfully port the TizenRT PM subsystem to your new hardware, enabling significant power savings for your applications.
