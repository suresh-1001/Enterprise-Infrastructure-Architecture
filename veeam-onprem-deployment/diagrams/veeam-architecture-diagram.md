```mermaid
graph TD
    VC[Veeam Backup Server] --> VP[Veeam Proxy]
    VP --> VR[Primary Repository]
    VR --> BCJ[Backup Copy Job]
    BCJ --> DR[Offsite / Immutable Repo]
    VC --> NA[NetApp Storage]
    NA --> SM[SnapMirror Target]
