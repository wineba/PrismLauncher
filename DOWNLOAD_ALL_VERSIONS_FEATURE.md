# Download All Minecraft Versions Feature

This document describes the implementation of a new settings feature that allows users to download all available Minecraft versions, their libraries, assets, and modloaders.

## Overview

This feature adds a button to the settings interface that initiates a bulk download of:
- All Minecraft versions
- Associated libraries for each version
- Game assets for each version
- Modloader files (Forge, Fabric, Quilt, etc.)

## Architecture

### 1. New Task Class: `BulkVersionDownloadTask`

**File:** `launcher/minecraft/download/BulkVersionDownloadTask.h` and `.cpp`

This extends `Task` and manages the overall download process:

```cpp
class BulkVersionDownloadTask : public Task {
    Q_OBJECT
public:
    BulkVersionDownloadTask(MinecraftInstance* instance);
    
protected:
    void executeTask() override;
    
private slots:
    void onVersionDownloadProgress();
    void onVersionDownloadFinished();
    void onVersionDownloadFailed(QString reason);
    
private:
    void downloadVersionManifest();
    void downloadVersion(const QString& version);
    void downloadLibrariesForVersion(const VersionFilePtr& versionFile);
    void downloadAssetsForVersion(const QString& assetIndex);
    void downloadModloaders();
    
    MinecraftInstance* m_instance;
    NetJob::Ptr m_currentJob;
    QStringList m_pendingVersions;
    int m_completedVersions = 0;
};
```

### 2. Settings Page UI Update

**File:** `launcher/ui/pages/instance/InstanceSettingsPage.ui`

Add a new button to the instance settings:
```xml
<widget class="QPushButton" name="downloadAllVersionsBtn">
    <property name="text">
        <string>Download All Versions</string>
    </property>
    <property name="toolTip">
        <string>Download all Minecraft versions, libraries, assets, and modloaders</string>
    </property>
</widget>
```

### 3. Settings Page Implementation

**File:** `launcher/ui/pages/instance/InstanceSettingsPage.h` and `.cpp`

Add handler for the new button:

```cpp
class InstanceSettingsPage : public MinecraftSettingsWidget, public BasePage {
    Q_OBJECT
    
private slots:
    void on_downloadAllVersionsBtn_clicked();
    void onBulkDownloadProgress(qint64 current, qint64 total);
    void onBulkDownloadFinished();
};
```

### 4. Global Settings Page Alternative

**File:** `launcher/ui/pages/global/LauncherPage.h` and `.cpp`

For downloading globally (not instance-specific), add similar UI and handler.

## Implementation Details

### Version Discovery

1. Fetch the version manifest from Mojang's servers
2. Parse available versions
3. Queue each version for download

### Library Management

1. For each version, extract library references from the version JSON
2. Download each library file to the appropriate cache location
3. Validate checksums if available

### Asset Management

1. Fetch the asset index for each version
2. Download assets from Mojang's CDN
3. Organize assets by version

### Modloader Support

1. Query popular modloader repositories:
   - Forge: Load metadata from Forge's version JSON
   - Fabric: Fetch from fabric-meta server
   - Quilt: Fetch from Quilt meta server
   - NeoForge: Fetch from NeoForge repositories

## File Structure

```
launcher/
├── minecraft/
│   ├── download/
│   │   ├── BulkVersionDownloadTask.h
│   │   ├── BulkVersionDownloadTask.cpp
│   │   ├── ModloaderDownloadTask.h
│   │   └── ModloaderDownloadTask.cpp
│   └── Library.h (existing - may need updates)
├── ui/
│   └── pages/
│       ├── instance/
│       │   ├── InstanceSettingsPage.h
│       │   ├── InstanceSettingsPage.cpp
│       │   └── InstanceSettingsPage.ui
│       └── global/
│           ├── LauncherPage.h
│           ├── LauncherPage.cpp
│           └── LauncherPage.ui
└── CMakeLists.txt (update to include new files)
```

## UI/UX Considerations

1. **Progress Dialog**: Show a detailed progress dialog with:
   - Current version being downloaded
   - Overall progress percentage
   - Estimated time remaining
   - Ability to pause/resume
   - Ability to cancel

2. **Background Operation**: Option to run in background with system notifications

3. **Storage Location**: 
   - Use existing library cache location
   - Show space required vs available
   - Warn if insufficient disk space

4. **Download Strategy**:
   - Parallel downloads (respecting bandwidth limits)
   - Retry failed downloads
   - Resume incomplete downloads

## Configuration

Add settings in `settings/SettingsObject`:
- `BulkDownloadConcurrent`: Number of concurrent downloads (default: 4)
- `BulkDownloadBandwidthLimit`: Bandwidth limit in KB/s (0 = unlimited)
- `DownloadAllVersionsOnStartup`: Boolean to download on launcher startup
- `LastBulkDownloadDate`: Timestamp of last bulk download

## Integration Points

### Existing Classes to Use

1. **NetJob**: For managing concurrent downloads
2. **Net::Download**: For individual file downloads
3. **HttpMetaCache**: For caching downloaded files
4. **Task**: Base class for async operations
5. **ProgressDialog**: For showing download progress

### Signals/Slots

```cpp
// Progress updates
void stepProgress(qint64 current, qint64 total);
void status(QString status);

// Completion
void succeeded();
void failed(QString reason);
void aborted();
```

## Testing Considerations

1. Test with various network speeds
2. Test with limited disk space scenarios
3. Test cancellation at various stages
4. Test with corrupted/invalid downloads
5. Test modloader availability across regions
6. Test with instance-specific vs global downloads

## Future Enhancements

1. **Selective Downloading**: Allow users to choose specific versions/modloaders
2. **Scheduling**: Schedule downloads for off-peak hours
3. **Delta Sync**: Only download changed/new versions
4. **Compression**: Compress downloads for faster transfer
5. **CDN Integration**: Use multiple CDN endpoints for faster downloads

## Security Considerations

1. Verify checksums for all downloads
2. Use HTTPS for all connections
3. Validate version manifests from trusted sources
4. Check file permissions and ownership
