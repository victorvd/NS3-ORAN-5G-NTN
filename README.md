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
### **1. Add Satellite Channel Model (NTN-specific)**  
Create `src/oran/model/ntn-channel.cc`:  
```cpp
#include "ns3/ntn-channel.h"
#include "ns3/log.h"

namespace ns3 {
NS_LOG_COMPONENT_DEFINE ("NtnChannel");

NtnChannel::NtnChannel () {
  // Initialize NTN parameters (delay, Doppler, etc.)
}

// Implement satellite propagation models
} // namespace ns3
```

### **2. Create Satellite Mobility Model**  
Create `src/oran/model/ntn-mobility.cc`:  
```cpp
#include "ns3/ntn-mobility.h"
#include "ns3/simulator.h"

namespace ns3 {

NtnMobilityModel::NtnMobilityModel () {
  // Initialize orbital parameters (LEO/MEO)
}

Vector NtnMobilityModel::DoGetPosition () const {
  // Implement SGP4 or simplified orbital mechanics
  return Vector(x, y, z);
}
} // namespace ns3
```

### **3. Modify NR Module for NTN**  
Edit `src/nr/model/nr-phy.cc` to add:  
```cpp
// Add NTN-specific PHY adaptations
void NrPhy::SetNtnMode (bool ntnEnabled) {
  m_ntnMode = ntnEnabled;
  // Adjust for long delays if enabled
}
```

### **4. Update O-RAN Interfaces**  
Edit `src/oran/model/oran-near-rt-ric.cc`:  
```cpp
void OranNearRtRic::ProcessNtnReport (Ptr<NtnReport> report) {
  // Handle satellite-specific metrics
  if (report->GetNodeType() == NTN_SATELLITE) {
    // Adjust scheduling for long RTT
  }
}
```

### **5. Update Build Files**  
Edit `src/oran/wscript`:  
```python
module.source.append('model/ntn-channel.cc')
module.source.append('model/ntn-mobility.cc')
```

### **Rebuild Everything**  
```bash
cd ~/ns3-install/ns-3-dev
./ns3 configure --enable-examples --enable-tests
./ns3 build
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

**Contribute back to the community!** 🛰️  

---

🔗 **References**:  
- [NS-3 Documentation](https://www.nsnam.org/documentation/)  
- [NIST ns3-oran](https://github.com/usnistgov/ns3-oran)  
- [CTTC NR Module](https://gitlab.com/cttc-lena/nr)  
- [3GPP NTN TR 38.811](https://www.3gpp.org/ftp/Specs/archive/)  
