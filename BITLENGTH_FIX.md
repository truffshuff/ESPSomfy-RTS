# Fix for Mixed 56-bit and 80-bit Devices

## Problem
When using both 56-bit motors and 80-bit wind sensors together, the system was unable to properly configure them because linked remotes (like wind sensors) were inheriting the global transceiver bit length setting instead of having their own independent bit length configuration.

## Solution
This fix allows each linked remote to have its own `bitLength` property, enabling you to configure:
- Your motor as 56-bit
- Your wind sensor as 80-bit

Both can now coexist and work properly on the same shade.

## Changes Made

### 1. Configuration File (ConfigFile.cpp)
- **Version bumped**: 24 → 25
- **Record size increased**: 276 → 283 bytes (to store bitLength for each of 7 linked remotes)
- **Save**: Each linked remote's `bitLength` is now saved to the configuration file
- **Load**: Each linked remote's `bitLength` is loaded from the configuration file
  - If `bitLength` is 0 or not set, it inherits from the shade's `bitLength` (backward compatible)

### 2. API Updates (Somfy.cpp, Somfy.h, Web.cpp)
- **linkRemote()**: Now accepts an optional `bitLength` parameter
  - Signature: `bool linkRemote(uint32_t remoteAddress, uint16_t rollingCode = 0, uint8_t bitLength = 0)`
  - If `bitLength` is 0, the linked remote inherits the shade's `bitLength`
  - If `bitLength` is provided (56 or 80), it uses that value

- **toJSON()**: Linked remotes now include their `bitLength` in JSON output

### 3. Web API (Web.cpp)
- The link remote API now accepts an optional `bitLength` field in the JSON payload

## How to Use

### Option 1: Via Web API
When linking your wind sensor to a shade, include the `bitLength` field:

```json
POST /linkRemote
{
  "shadeId": 1,
  "remoteAddress": 123456,
  "rollingCode": 0,
  "bitLength": 80
}
```

### Option 2: Manual Configuration
1. After upgrading to this version, the config file will be automatically upgraded to version 25
2. All existing linked remotes will inherit their shade's `bitLength` (backward compatible)
3. To manually set a linked remote's bitLength to 80:
   - Use the web API as shown above
   - Or modify the configuration through the web interface (if bitLength field is added to the UI)

### Option 3: Serial Monitor
After linking a remote, you can verify its bitLength by checking the JSON output.

## Migration Path

### For Existing Users:
1. **Backup your configuration** before upgrading
2. Flash the new firmware
3. On first boot, the system will:
   - Detect config version 24
   - Upgrade to version 25
   - Set all linked remotes' `bitLength` to match their parent shade's `bitLength`
4. Re-link your wind sensor with `bitLength: 80` to override the default

### For New Users:
- When linking a wind sensor, specify `bitLength: 80` in the API call
- When linking a regular remote, omit `bitLength` or set it to match your motor (usually 56)

## Technical Details

### Inheritance Rules:
1. **New linked remote**: If `bitLength` is not specified when linking, inherits from shade
2. **Existing linked remote**: If re-linked with `bitLength`, updates to the new value
3. **Loading from config**: If stored `bitLength` is 0, inherits from shade

### Backward Compatibility:
- Config version < 25: All linked remotes inherit from shade (old behavior)
- Config version >= 25: Each linked remote has its own `bitLength`
- API calls without `bitLength` parameter: Linked remote inherits from shade

## Example Scenario

You have:
- A 56-bit motor shade
- An 80-bit wind sensor

### Setup:
1. Set your transceiver type to 56-bit (for the motor)
2. Configure your shade as 56-bit
3. Link your wind sensor with `bitLength: 80`:
```json
{
  "shadeId": 1,
  "remoteAddress": 0xABCDEF,
  "bitLength": 80
}
```

Now:
- Motor commands will be sent as 56-bit frames
- Wind sensor commands will be sent as 80-bit frames
- Both devices work correctly on the same shade!

## Files Modified
- `ConfigFile.cpp`: Version bump, save/load bitLength for linked remotes
- `ConfigFile.h`: (if applicable)
- `Somfy.cpp`: Updated linkRemote(), toJSON()
- `Somfy.h`: Updated linkRemote() signature
- `Web.cpp`: Updated linkRemote API to accept bitLength

## Testing
1. Verify motor still works (56-bit)
2. Link wind sensor with bitLength=80
3. Send sensor command - should transmit as 80-bit
4. Send motor command - should transmit as 56-bit
5. Check JSON output to verify each remote's bitLength is correct
