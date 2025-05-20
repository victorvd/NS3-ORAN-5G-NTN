# **Tutorial: Installing NS-3 with ns3-oran and CTTC NR for 5G NTN Support**  
*A complete guide to set up NS-3 with both the official `ns3-oran` module and CTTC's 5G NR module for Non-Terrestrial Networks (NTN) simulations.*

---

## **Prerequisites**  
- **Ubuntu 20.04/22.04** (recommended)  
- **Minimum 15GB disk space** (NR module is large)  
- **8GB RAM recommended**  
- **Git, GCC, Python3, CMake, and dependencies**  

### **1. Install Dependencies**  
```bash
sudo apt update
sudo apt install -y g++ python3 python3-dev pkg-config sqlite3 cmake python3-setuptools git qt5-default gir1.2-goocanvas-2.0 python3-gi python3-gi-cairo python3-pygraphviz gir1.2-gtk-3.0 ipython3 openmpi-bin openmpi-common openmpi-doc libopenmpi-dev autoconf cvs bzr unrar gsl-bin libgsl-dev libgslcblas0 wireshark tcpdump libxml2 libxml2-dev libboost-all-dev libasio-dev
```

---

## **Step 1: Install NS-3**  
### **Clone and Build NS-3**  
```bash
mkdir ~/ns3-install && cd ~/ns3-install
git clone https://gitlab.com/nsnam/ns-3-dev.git
cd ns-3-dev
./ns3 configure --enable-examples --enable-tests
./ns3 build
```
**Verify Installation:**  
```bash
./test.py
```

---

## **Step 2: Install ns3-oran (NIST Official Version)**  
### **Clone and Integrate ns3-oran**  
```bash
cd ~/ns3-install
git clone https://github.com/usnistgov/ns3-oran.git
cp -r ns3-oran ns-3-dev/src/oran  # Copy and rename to 'oran'
```

---

## **Step 3: Install CTTC NR Module**  
### **Clone and Integrate NR Module**  
```bash
cd ~/ns3-install
git clone https://gitlab.com/cttc-lena/nr.git
cp -r nr ns-3-dev/src/
```

### **Rebuild NS-3 with Both Modules**  
```bash
cd ~/ns3-install/ns-3-dev
./ns3 configure --enable-examples --enable-tests
./ns3 build
```

### **Verify Correct Integration**  
After editing:
```bash
cd ~/ns3-install/ns-3-dev
./ns3 configure --enable-examples --enable-tests | grep -E "oran|nr"
```

You should see:
```bash
Processing src/oran
Processing src/nr
```

---

### **Important Notes**

#### 1. No Need for Explicit add_subdirectory
The existing loop already handles this automatically for all modules in all_modules_in.

#### 2. If Modules Don't Appear
 - Verify both folders, `oran` and `nr`, contain CMakeLists.txt

#### 3. Clean Build Recommended
```bash
./ns3 clean
./ns3 configure --enable-examples --enable-tests
./ns3 build
```

---

## **Step 4: Extend for 5G NTN Support**  

### **1. Directory Structure Overview**
```
ns3-dev/
└── src/
    ├── oran/
    │   ├── model/          # For new NTN files
    │   ├── examples/       # For test scenarios
    │   └── wscript         # To modify
    └── nr/
        └── model/          # For NR modifications
```

### **2. New Files to Create**

#### **A. NTN Channel Model**
```bash
# Navigate to NS-3's oran module
cd ~/ns3-install/ns-3-dev/src/oran/model

# Create new channel model files
touch ntn-channel.h ntn-channel.cc
```

```bash
// src/oran/model/ntn-channel.h
#include "ns3/propagation-loss-model.h"
#include "ns3/propagation-delay-model.h"

namespace ns3 {
class NtnChannel : public Object {
public:
  static TypeId GetTypeId();
  NtnChannel();
  
  // 3GPP TR 38.811 compliant models
  double CalculatePathLoss(Ptr<MobilityModel> a, Ptr<MobilityModel> b) const;
  Time CalculateDelay(Ptr<MobilityModel> a, Ptr<MobilityModel> b) const;

private:
  double m_frequency; // Carrier frequency in Hz
  double m_weatherLoss; // Atmospheric attenuation
};
} // namespace ns3
```

```bash
// src/oran/model/ntn-channel.cc
#include "ns3/ntn-channel.h"
#include "ns3/log.h"

namespace ns3 {
NS_LOG_COMPONENT_DEFINE ("NtnChannel");

NtnChannel::NtnChannel() {
  // Initialize with LEO default parameters
  m_frequency = 2e9; // 2 GHz
  m_weatherLoss = 0.1; // dB/km
}

double NtnChannel::CalculatePathLoss(Ptr<MobilityModel> a, Ptr<MobilityModel> b) const {
  // Implement 3GPP TR 38.811 model
  return fspl + atmosphericLoss + rainAttenuation;
}
} // namespace ns3
```


#### **B. Satellite Mobility Model**
```bash
# In the same directory /ns3-install/ns-3-dev/src/oran/model
touch ntn-mobility.h ntn-mobility.cc
```

```bash
// src/oran/model/ntn-mobility.h
#include "ns3/mobility-model.h"

namespace ns3 {
class NtnMobilityModel : public MobilityModel {
public:
  static TypeId GetTypeId();
  NtnMobilityModel();
  
private:
  virtual Vector DoGetPosition() const;
  virtual Vector DoGetVelocity() const;
  
  // Orbital parameters
  double m_altitude; // km
  double m_inclination; // degrees
};
} // namespace ns3
```

```bash
// src/oran/model/ntn-mobility.cc
#include "ns3/ntn-mobility.h"
#include "ns3/sgp4.h" // For precise orbital mechanics

namespace ns3 {

NtnMobilityModel::NtnMobilityModel() {
  // Initialize LEO parameters
  m_altitude = 1200; // km
  m_inclination = 53; // degrees
}

Vector NtnMobilityModel::DoGetPosition() const {
  // Implement SGP4 or simplified circular orbit
  double t = Simulator::Now().GetSeconds();
  return CalculateLEOPosition(t, m_altitude, m_inclination);
}
} // namespace ns3
```

#### **C. Example Scenario (Optional)**
```bash
cd ~/ns3-install/ns-3-dev/src/oran/examples/oran-ntn-example.cc
touch oran-ntn-example.cc
```
---

### **3. Existing Files to Modify**

#### **A. NR PHY Layer (5G NR Adaptations)**
```bash
cd ~/ns3-install/ns-3-dev/src/nr/model
nano nr-phy.h nr-phy.cc
```

Add the MobilityModel class declaration. Add this include at the top of the file (with other includes):
```bash
#include "ns3/mobility-model.h"
```

Add NTN mode and Doppler compensation
```bash
// src/nr/model/nr-phy.h
// Add to the NrPhy class declaration (around line 200-300 in most versions)
public:
  /**
   * Enable/Disable NTN mode
   * @param ntnEnabled true for NTN operation
   */
  void SetNtnMode(bool ntnEnabled);

  /**
   * Set NTN Doppler parameters
   * @param ueMobility UE mobility model
   * @param satMobility Satellite mobility model
   */

    // after this line "std::list<Ptr<NrControlMessage>> m_ctrlMsgs;"
    bool m_ntnMode {false}; //!< NTN operation flag
```

```bash
// src/nr/model/nr-phy.cc
void // Add anywhere in the implementation (around line 1000-1500 in most versions)
NrPhy::SetNtnMode(bool ntnEnabled)
{
    m_ntnMode = ntnEnabled;
    if (ntnEnabled) {
        m_tddPattern = GetNtnTddPattern();
        m_harqTimer = GetNtnHarqTimer(); // Now properly declared
        
        NS_LOG_INFO("Configured NTN mode with " << 
                   m_tddPattern.size() << "-slot pattern and " <<
                   m_harqTimer.GetMilliSeconds() << "ms HARQ timer");
    }
}

void
NrPhy::DoSetNtnDopplerParameters(Ptr<MobilityModel> ueMobility,
                                Ptr<MobilityModel> satMobility)
{
  NS_LOG_FUNCTION(this);
  
  // Calculate relative velocity vector
  Vector3D relVelocity = ueMobility->GetVelocity() - satMobility->GetVelocity();
  
  // Project onto line-of-sight vector
  Vector3D losVector = satMobility->GetPosition() - ueMobility->GetPosition();
  double radialVelocity = relVelocity.x * losVector.x + 
                         relVelocity.y * losVector.y +
                         relVelocity.z * losVector.z;
  radialVelocity /= losVector.GetLength();
  
  // Calculate Doppler shift (f_d = (v/c)*f_c)
  double dopplerShift = (radialVelocity / 3e8) * m_centerFrequency;
  
  // Apply pre-compensation
  m_txFrequency = m_centerFrequency - dopplerShift;
  NS_LOG_DEBUG("Doppler pre-compensation applied: " << dopplerShift << " Hz");
  
  UpdateTxConfiguration();
}
```

#### **B. O-RAN RIC (Interface Extensions)**
```bash
cd ~/ns3-install/ns-3-dev/src/oran/model
nano oran-report.h oran-near-rt-ric.cc
```

Add NTN report processing
```bash
// src/oran/model/oran-report.h
struct NtnBeamReport {
  uint16_t beamId;
  double snr;
  Time propagationDelay;
  Vector3d satellitePosition;
};

// src/oran/model/oran-near-rt-ric.cc
void OranNearRtRic::ProcessNtnReport(Ptr<NtnReport> report) {
  if (report->nodeType == NTN_SATELLITE) {
    // Special handling for satellite links
    m_ntnScheduler->AdjustForDelay(report->propagationDelay);
    
    if (NeedBeamHandover(report)) {
      TriggerNtnHandover(report->beamId);
    }
  }
}
```

#### **C. Additional Required Changes**

For full functionality, you'll need to add these to `src/nr/model/nr-phy.h` (in the class declaration):
```bash
class NrPhy {

// ... existing code ...
public:
    // Additions
    typedef std::vector<LteNrTddSlotType> TddPattern;
    TddPattern GetNtnTddPattern() const;
    Time GetNtnHarqTimer() const;
    void SetNtnMode(bool ntnEnabled);
    void DoSetNtnDopplerParameters(Ptr<MobilityModel> ueMobility,
                               Ptr<MobilityModel> satMobility);
    void UpdateTxConfiguration();

private:
    // Additions
    // bool m_ntnMode {false};
    Time m_harqTimer;
    TddPattern m_tddPattern;
    double m_centralFrequency;
    double m_txFrequency;

// ... rest of class ...
};
```

Define `TddPattern` adding this to `nr-phy.h` just after the header includes but before the `NrPhy class` declaration::
```bash
// src/nr/model/nr-phy.h

// after "namespace ns3 {"

// Add these definitions before any class declarations
enum class TddSlotType {
    D, // Downlink
    U, // Uplink
    G  // Guard period
};

struct TddPattern {
    std::vector<TddSlotType> slots;
    uint16_t numSlots;
};

// before "class NrPhy : public Object {"
```


Add these implementations to `src/nr/model/nr-phy.cc`:
```bash
// New implementations
NrPhy::TddPattern
NrPhy::GetNtnTddPattern() const  // Note the NrPhy:: prefix
{
    return {
        LteNrTddSlotType::DL,  // Downlink
        LteNrTddSlotType::DL,
        LteNrTddSlotType::S,    // Special/Guard
        LteNrTddSlotType::UL,   // Uplink
        LteNrTddSlotType::UL,
        LteNrTddSlotType::S     // Special/Guard
    };
}

Time
NrPhy::GetNtnHarqTimer() const  // Note the NrPhy:: prefix
{
    // Adjust these values based on your NTN requirements:
    if (m_ntnMode) {
        // LEO satellites: 100-200ms
        // GEO satellites: 500-600ms
        return MilliSeconds(150); // Example for LEO
    }
    return MilliSeconds(8); // Default terrestrial value
}

void NrPhy::UpdateTxConfiguration() {
    if (m_spectrumPhy) {
        m_spectrumPhy->SetTxFrequency(m_txFrequency);
    }
    NS_LOG_INFO("Updated TX config. Frequency: " << m_txFrequency << " Hz");
}
```

Be sure this enum is present in `src/nr/model/nr-control-messages.h`:
```bash
// Ensure this enum exists
enum LteNrTddSlotType : uint8_t {
    DL = 0, // Downlink
    S = 1,  // Special/Guard
    F = 2,  // Flexible
    UL = 3  // Uplink
};
```

Check compilation:
```bash
cd ~/ns3-install/ns-3-dev
./ns3 build nr
```

#### **D. Build System (wscript)**
```bash
~/ns3-install/ns-3-dev/src/oran/
nano wscript
```

Register new `ntn-channel.cc` and `ntn-mobility.cc`
```bash
module.source.append('model/ntn-channel.cc')
module.source.append('model/ntn-mobility.cc')
```

---

### **4. Detailed Path Summary**

| **Action**       | **File Path**                          | **Purpose**                          |
|------------------|---------------------------------------|--------------------------------------|
| **Create**       | `src/oran/model/ntn-channel.h`        | NTN propagation loss/delay models    |
| **Create**       | `src/oran/model/ntn-channel.cc`       | Implementation                       |
| **Create**       | `src/oran/model/ntn-mobility.h`       | Satellite orbital models             |
| **Create**       | `src/oran/model/ntn-mobility.cc`      | Implementation                       |
| **Modify**       | `src/nr/model/nr-phy.h`               | Add `SetNtnMode()` declaration       |
| **Modify**       | `src/nr/model/nr-phy.cc`              | Implement NTN adaptations            |
| **Modify**       | `src/oran/model/oran-near-rt-ric.h`   | Add NTN report handling              |
| **Modify**       | `src/oran/model/oran-near-rt-ric.cc`  | Implement NTN logic                  |
| **Modify**       | `src/oran/wscript`                    | Register new source files            |

---

### **5. Verification Commands**
After creating/modifying files:

```bash
# Check new files exist
ls src/oran/model/ntn-* 

# Verify builds
cd ~/ns3-install/ns-3-dev
./ns3 configure --enable-examples --enable-tests | grep "oran"
./ns3 build

# Run test (if example created)
./ns3 run src/oran/examples/oran-ntn-example
```

---

## **Step 5: Test 5G NTN Scenario**  
Create `scratch/ntn-demo.cc`:  
```cpp
#include "ns3/core-module.h"
#include "ns3/oran-module.h"
#include "ns3/nr-module.h"

using namespace ns3;

int main (int argc, char *argv[]) {
  // Create satellite gNB
  NodeContainer satellite;
  satellite.Create(1);
  
  // Install NTN mobility
  Ptr<NtnMobilityModel> mob = CreateObject<NtnMobilityModel>();
  satellite.Get(0)->AggregateObject(mob);

  // Configure NR for NTN
  Ptr<NrGnbNetDevice> satGnb = CreateObject<NrGnbNetDevice>();
  satGnb->GetPhy()->SetNtnMode(true);

  // Connect to O-RAN RIC
  Ptr<OranNearRtRic> ric = CreateObject<OranNearRtRic>();
  ric->AddNtnNode(satGnb);

  Simulator::Run();
  Simulator::Destroy();
  return 0;
}
```
**Run the Demo:**  
```bash
./ns3 run scratch/ntn-demo
```

---

## **Troubleshooting**  
- **Build Fails**:  
  - Verify `ns3-oran` and `nr` are compatible with your NS-3 version  
  - Check `./ns3 configure` output for missing dependencies  
- **Runtime Errors**:  
  - For long delays, set:  
    ```cpp
    GlobalValue::Bind("SchedulerType", StringValue("ns3::CalendarScheduler"));
    ```

---

## **Next Steps**  
1. **Add 3GPP NTN channel models** (TR 38.811)  
2. **Implement Doppler pre-compensation** in PHY layer  
3. **Extend O-RAN E2 interfaces** for dynamic beam management  

---

🔗 **References**:  
- [NS-3 Documentation](https://www.nsnam.org/documentation/)  
- [NIST ns3-oran](https://github.com/usnistgov/ns3-oran)  
- [CTTC NR Module](https://gitlab.com/cttc-lena/nr)  
- [3GPP NTN TR 38.811](https://www.3gpp.org/ftp/Specs/archive/)  
