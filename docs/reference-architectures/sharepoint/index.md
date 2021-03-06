---
title: 在 Azure 中執行高可用性的 SharePoint Server 2016 伺服器陣列
description: 在 Azure 上設定高可用性 SharePoint Server 2016 伺服器陣列的作法已經過驗證。
author: njray
ms.date: 08/01/2017
ms.openlocfilehash: d1e3f0b73c94844ac649bf2abb6917809202fdb7
ms.sourcegitcommit: c441fd165e6bebbbbbc19854ec6f3676be9c3b25
ms.translationtype: HT
ms.contentlocale: zh-TW
ms.lasthandoff: 03/30/2018
---
# <a name="run-a-high-availability-sharepoint-server-2016-farm-in-azure"></a>在 Azure 中執行高可用性的 SharePoint Server 2016 伺服器陣列

此參考架構會示範一組經過驗證的作法，也就是使用 MinRole 拓樸和 SQL Server Always On 可用性群組，在 Azure 上設定高可用性的 SharePoint Server 2016 的伺服器陣列。 SharePoint 伺服器陣列會部署在受保護且沒有網際網路對應端點或空間的虛擬網路。 [**部署這個解決方案**。](#deploy-the-solution) 

![](./images/sharepoint-ha.png)

下載這個架構的 [Visio 檔案][visio-download]。

## <a name="architecture"></a>架構

此架構是根據[執行適用於多層式架構應用程式的 Windows 虛擬機器][windows-n-tier]建置的。 它會將具有高可用性的 SharePoint Server 2016 伺服器陣列部署在 Azure 虛擬網路 (VNet) 中。 此架構適用於測試或生產環境，以及搭配 Office 365 的 SharePoint 混合式基礎結構，或是當做災害復原案例的基礎。

此架構由下列元件組成：

- **資源群組**。 [資源群組][resource-group]是保存 Azure 相關資源的容器。 一個資源群組會用於 SharePoint 伺服器，而另一個資源群組會用於虛擬網路和負載平衡器等不依附虛擬機器的基礎結構元件。

- **虛擬網路 (VNet)**。 虛擬機器會部署在具有唯一內部網路位址空間的 VNet 中。 VNet 會再細分為子網路。 

- **虛擬機器 (VM)**。 虛擬機器會部署至 VNet，而所有虛擬機器都會有指派的私人靜態 IP 位址。 靜態 IP 位址建議用於執行 SQL Server 和 SharePoint Server 2016 的虛擬機器，以避免在重新啟動後發生 IP 位址快取問題和位址變更的情形。

- **可用性集合**。 將每個 SharePoint 角色的虛擬機器放在個別的[可用性集合][availability-set]中，然後為每個角色佈建至少兩部虛擬機器 (VM)。 這讓虛擬機器能夠符合較高服務等級協定 (SLA) 的資格。 

- **內部負載平衡器**。 [負載平衡器][load-balancer]會將 SharePoint 要求流量從內部部署網路散發至 SharePoint 伺服器陣列的前端 Web 伺服器。 

- **網路安全性群組 (NSG)**。 系統會為包含虛擬機器的每個子網路，建立[網路安全性群組][nsg]。 使用 NSG 來限制 VNet 內的網路流量，以區隔子網路。 

- **閘道**。 閘道會提供內部部署網路與 Azure 虛擬網路之間的連線。 您的連線可以使用 ExpressRoute 或站對站 VPN。 如需詳細資訊，請參閱[將內部部署網路連線至 Azure][hybrid-ra]。

- **Windows Server Active Directory (AD) 網域控制站**。 由於 SharePoint Server 2016 不支援使用 Azure Active Directory Domain Services，因此您必須部署 Windows Server AD 網域控制站。 這些網域控制站會在 Azure VNet 中執行，並與內部部署 Windows Server AD 樹系之間有信任關係。 SharePoint 伺服器陣列資源的用戶端 Web 要求會在 VNet 中進行驗證 ，而不是透過內部部署網路的閘道連線傳送該驗證流量。 DNS 中會建立內部網路 A 或 CNAME 記錄，讓內部網路使用者可以將 SharePoint 伺服器陣列的名稱解析為內部負載平衡器的私用 IP 位址。

- **SQL Server Always On 可用性群組**。 若要取得 SQL Server 資料庫的高可用性使用，我們建議您使用 [可用性群組][sql-always-on]。 用於 SQL Server 的虛擬機器有兩個。 一個包含主要資料庫複本，另一個包含次要複本。 

- **多數節點 VM**。 此虛擬機器可讓您為容錯移轉叢集建立仲裁。 如需詳細資訊，請參閱[了解容錯移轉叢集中的仲裁設定][sql-quorum]。

- **SharePoint 伺服器**。 SharePoint 伺服器會執行 Web 前端、快取、應用程式和搜尋角色。 

- **Jumpbox**。 也稱為[防禦主機][bastion-host]。 這是網路上系統管理員用來連線到其他 VM 的安全 VM。 Jumpbox 具有 NSG，只允許來自安全清單上公用 IP 位址的遠端流量。 NSG 應該允許遠端桌面 (RDP) 流量。

## <a name="recommendations"></a>建議

您的需求可能和此處所述的架構不同。 請使用以下建議作為起點。

### <a name="resource-group-recommendations"></a>資源群組建議

我們建議您根據伺服器角色來區隔資源群組，並且具有屬於基礎結構元件 (全域資源) 的個別資源群組。 在此架構中，SharePoint 資源會組成一個群組，而 SQL Server 和其他公用程式資產會組成另一個群組。

### <a name="virtual-network-and-subnet-recommendations"></a>虛擬網路和子網路建議

針對每個 SharePoint 角色使用一個子網路，另有一個屬於閘道的子網路，和另一個用於 jumpbox 的子網路。 

閘道子網路必須命名為 *GatewaySubnet*。 從虛擬網路位址空間的最後一個部分，指定閘道子網路位址空間。 如需詳細資訊，請參閱[使用 VPN 閘道將內部部署網路連線至 Azure][hybrid-vpn-ra]。

### <a name="vm-recommendations"></a>VM 建議

以標準 DSv2 虛擬機器大小為基礎，此架構需要最少 38 個核心：

- Standard_DS3_v2 上 8 個 SharePoint 伺服器 (每部伺服器 4 個核心) = 32 個核心
- Standard_DS1_v2 上 2 個 Active Directory 網域控制站 (每個控制站 1 個核心) = 2 個核心
- Standard_DS1_v2 上 2 個 SQL Server 虛擬機器 = 2 個核心
- Standard_DS1_v2 上 1 個多數節點 = 1 個核心
- Standard_DS1_v2 上 1 個管理伺服器 = 1 個核心

核心總數將取決於您選取的虛擬機器大小。 如需詳細資訊，請參閱下方的 [SharePoint Server 建議](#sharepoint-server-recommendations)。

請確定您的 Azure 訂用帳戶具有足夠的虛擬機器核心配額可用於部署，否則部署將會失敗。 請參閱 [Azure 訂用帳戶和服務限制、配額與限制][quotas]。 
 
### <a name="nsg-recommendations"></a>NSG 建議

我們建議每個包含虛擬機器的子網路都要有一個 NSG，以啟用子網路隔離。 如果您想要設定子網路隔離，請新增 NSG 規則，此規則會定義每個子網路允許或拒絕的輸入或輸出流量。 如需詳細資訊，請參閱[使用網路安全性群組來篩選網路流量][virtual-networks-nsg]。 

請勿將 NSG 指派至閘道子網路，否則閘道將會停止運作。 

### <a name="storage-recommendations"></a>儲存體建議

伺服器陣列中的虛擬機器儲存體設定，應符合內部部署適用的最佳做法。 SharePoint 伺服器應有用於記錄的個別磁碟。 裝載搜尋索引角色的 SharePoint 伺服器需要額外的磁碟空間來儲存搜尋索引。 SQL Server 的標準作法是將資料和記錄分開。 為資料庫備份儲存體新增更多磁碟，並針對 [tempdb][tempdb] 使用個別磁碟。

為擁有最佳可靠性，我們建議您使用 [Azure 受控磁碟][managed-disks]。 受控磁碟可確保可用性設定組內的虛擬機器磁碟是各自獨立的，以避免發生單一失敗點。 

> [!NOTE]
> 目前，此參考架構的 Resource Manager 範本並未使用受控磁碟。 我們正計劃更新範本以使用受控磁碟。

針對所有 SharePoint 和 SQL Server 虛擬機器，請使用進階受控磁碟。 針對多數節點伺服器、網域控制站和管理伺服器，您可以使用標準受控磁碟。 

### <a name="sharepoint-server-recommendations"></a>SharePoint Server 建議

設定 SharePoint 伺服器陣列之前，請確定您的每個服務都有一個 Windows Server Active Directory 帳戶。 針對此架構，您至少需要下列網域等級的帳戶來隔離每個角色的權限：

- SQL Server 服務帳戶
- 設定使用者帳戶
- 伺服器陣列帳戶
- 搜尋服務帳戶
- 內容存取帳戶
- Web 應用程式集區帳戶
- 服務應用程式集區帳戶
- 快取進階使用者帳戶
- 快取進階讀者帳戶

對於除了搜尋索引子以外的所有角色，我們建議使用 [Standard_DS3_v2][vm-sizes-general] 虛擬機器大小。 搜尋索引子應至少使用 [Standard_DS13_v2][vm-sizes-memory]大小。 

> [!NOTE]
> 針對搜尋索引子，此參考架構的 Resource Manager 範本會使用較小的 DS3 大小，以便用於測試和部署。 對於生產部署，請使用 DS13 或更大的大小。 

對於生產工作負載，請參閱[適用於 SharePoint Server 2016 的硬體和軟體需求][sharepoint-reqs]。 

若要符合磁碟輸送量最低每秒 200 MB 的支援需求，請務必規劃搜尋架構。 請參閱[在 SharePoint Server 2013 中的規劃企業搜尋架構][sharepoint-search]。 也請遵循[在 SharePoint Server 2016 中進行編目的最佳做法][sharepoint-crawling]。

此外，請將搜尋元件資料存放在高效能的個別儲存磁碟區或分割區。 若要減少負載並提高輸送量，請設定此架構所需的物件快取使用者帳戶。 將 Windows Server 作業系統檔案、SharePoint Server 2016 程式檔和診斷記錄分散在三個一般效能的個別儲存磁碟區或分割區。 

如需有關這些建議的詳細資訊，請參閱[在 SharePoint Server 2016 中初始部署管理與服務帳戶][sharepoint-accounts]。

### <a name="hybrid-workloads"></a>混合式工作負載

此參考架構會部署可用來當作 [SharePoint 混合式環境][sharepoint-hybrid] &mdash; 的 SharePoint Server 2016 伺服器陣列，也就是將 SharePoint Server 2016 延伸為 Office 365 SharePoint Online。 如果您有 Office Online Server，請參閱 [Azure 中的 Office Web Apps 和 Office Online Server 支援][office-web-apps]。

此部署中的預設服務應用程式是專為支援混合式工作負載所設計。 所有 SharePoint Server 2016 和 Office 365 混合式工作負載，皆可在不變更 SharePoint 基礎結構的狀況下部署到此伺服器陣列，但有一個例外：雲端混合式搜尋服務應用程式不可部署到裝載現有搜尋拓撲的伺服器。 因此，必須將一個或多個以搜尋角色為基礎的虛擬機器新增至伺服器陣列，以支援此混合式案例。

### <a name="sql-server-always-on-availability-groups"></a>SQL Server Always On 可用性群組

此架構會使用 SQL Server 虛擬機器，因為 SharePoint Server 2016 無法使用 Azure SQL Database。 若要在 SQL Server 中支援高可用性，我們建議您使用 Always On 可用性群組，此可用性群組會指定一組一起進行容錯移轉的資料庫，使其具有高可用性和復原能力。 在此參考架構中，資料庫會在部署期間建立，但您必須手動啟用 Always On 可用性群組，並將 SharePoint 資料庫新增至可用性群組。 如需詳細資訊，請參閱[建立可用性群組並新增 SharePoint 資料庫][create-availability-group]。

我們也建議您將接聽程式 IP 位址新增到叢集，這是 SQL Server 虛擬機器內部負載平衡器的私人 IP 位址。

如需深入了解建議的 VM 大小和其他在 Azure 中執行的 SQL Server 效能建議，請參閱[Azure 虛擬機器中的 SQL Server 效能最佳做法][sql-performance]。 也請遵循 [SharePoint Server 2016 伺服器陣列中的 SQL Server 最佳作法][sql-sharepoint-best-practices]中的建議。

我們建議您將多數節點伺服器所在的電腦和複寫協力電腦分開。 伺服器會在高安全性模式工作階段中啟用次要的複寫協力電腦伺服器，以辨別是否啟動自動容錯移轉。 與兩個協力電腦不同的是，多數節點伺服器的服務目標不是資料庫，而是支援自動容錯移轉。 

## <a name="scalability-considerations"></a>延展性考量

若要相應增加現有伺服器，只需變更虛擬機器大小。 

透過 SharePoint Server 2016 中的 [MinRoles][minroles] 功能，您可以根據伺服器的角色擴充伺服器，也可從角色中移除伺服器。 當您將伺服器新增至角色上時，您可以指定任何單一角色或合併角色的其中一個。 不過，如果您將伺服器新增至搜尋角色，您也必須使用 PowerShell 重新設定搜尋拓撲。 您也可以使用 MinRoles 來轉換角色。 如需詳細資訊，請參閱[在 SharePoint Server 2016 中管理 MinRole 伺服器陣列][sharepoint-minrole]。

請注意，SharePoint Server 2016 不支援使用虛擬機器擴展集來進行自動調整。

## <a name="availability-considerations"></a>可用性考量

此參考架構支援 Azure 區域內的高可用性，因為每個角色至少有兩個虛擬機器部署在可用性設定組。

若要防止受到區域性失敗的影響，請在不同 Azure 區域中建立個別的災害復原伺服器陣列。 您的復原時間目標 (RTO) 和復原點目標 (RPO) 會決定安裝需求。 如需詳細資訊，請參閱[選擇適用於 SharePoint 2016 的災害復原策略][sharepoint-dr]。 次要區域應與主要區域是*配對的區域*。 若發生廣泛中斷事件，會優先復原所有配對中的一個區域。 如需詳細資訊，請參閱[商務持續性和災害復原 (BCDR)：Azure 配對的區域][paired-regions]。

## <a name="manageability-considerations"></a>管理性考量

若要操作和維護伺服器、伺服器陣列和站台，請遵循 SharePoint 作業的建議作法。 如需詳細資訊，請參閱 [SharePoint Server 2016 的作業][sharepoint-ops]。

在 SharePoint 環境中管理 SQL Server 時要考量的工作，可能會與一般管理資料庫應用程式要考量的工作不同。 最佳作法是以每晚累計的備份，在每週完整備份所有 SQL 資料庫。 每隔 15 分鐘備份一次交易記錄。 另一個做法是在資料庫上實作 SQL Server 維護工作，同時停用內建的 SharePoint 維護工作。 如需詳細資訊，請參閱[儲存體和 SQL Server 容量規劃和設定][sql-server-capacity-planning]。 

## <a name="security-considerations"></a>安全性考量

用來執行 SharePoint Server 2016 的網域層級服務帳戶，需要 Windows Server AD 網域控制站來進行網域加入及驗證處理程序。 Azure Active Directory Domain Services 無法用於此用途。 若要擴充已經在內部網路中就緒的 Windows Server AD 身分識別基礎結構，此架構會使用現有內部部署 Windows Server AD 樹系的兩個 Windows Server AD 複本網域控制站。

此外，規劃安全性強化絕對是個明智的選擇。 其他建議如下：

- 將規則新增至 NSG 以隔離子網路和角色。
- 不要將公用 IP 位址指派給虛擬機器。
- 針對入侵偵測和乘載分析，請考慮在前端 Web 伺服器之前使用網路虛擬設備，而不是使用內部 Azure 負載平衡器。
- 您可以選擇對伺服器之間的純文字傳輸加密使用 IPsec 原則。 如果您也進行了子網路隔離，請將您的網路安全性群組規則更新為允許 IPsec 流量。
- 安裝適用於虛擬機器的反惡意程式碼代理程式。

## <a name="deploy-the-solution"></a>部署解決方案

此參考架構的部署指令碼可在 [GitHub][github] 上取得。 

您可以以累加方式或一次全部部署此架構。 第一次，建議您使用累加式部署，以便您看到每個部署的執行方式。 使用下列其中一個 mode 參數來指定增量。

| Mode           | 作用                                                                                                            |
|----------------|-------------------------------------------------------------------------------------------------------------------------|
| onprem         | (選擇性) 部署模擬的內部部署網路環境以用於測試或評估。 這個步驟不會連線到實際的內部部署網路。 |
| infrastructure | 將 SharePoint 2016 網路基礎結構和 jumpbox 部署到 Azure。                                                |
| createvpn      | 部署 SharePoint 和內部部署網路的虛擬網路閘道並使其連線。 只有當您執行 `onprem` 步驟時，才須執行此步驟。                |
| workload       | 將 SharePoint 伺服器部署到 SharePoint 網路。                                                               |
| security       | 將網路安全性群組部署到 SharePoint 網路。                                                           |
| 所有            | 部署所有先前的部署。                            


若要在模擬的內部部署網路環境上以累加方式部署架構，請依序執行下列步驟：

1. onprem
2. infrastructure
3. createvpn
4. workload
5. security

若不是在模擬的內部部署網路環境上以累加方式部署架構，請依序執行下列步驟：

1. infrastructure
2. workload
3. security

若要在一個步驟中部署所有項目，請使用 `all`。 請注意，整個程序可能需費時數小時。

### <a name="prerequisites"></a>先決條件

* 安裝最新版的 [Azure PowerShell][azure-ps]。

* 在部署此參考架構之前，請確認您的訂用帳戶有足夠的配額 (至少 38 個核心)。 如果沒有足夠的配額，請使用 Azure 入口網站來提交支援要求以取得更多配額。

* 若要預估此部署的成本，請參閱 [Azure 定價計算機][azure-pricing]。

### <a name="deploy-the-reference-architecture"></a>部署參考架構

1.  將 [GitHub 存放庫][github]下載或複製到本機電腦。

2.  開啟 PowerShell 視窗並瀏覽至 `/sharepoint/sharepoint-2016` 資料夾。

3.  執行下列 PowerShell 命令。 針對\<subscription id\>，您的 Azure 訂用帳戶識別碼。 針對\<location\>，請指定 Azure 區域，例如 `eastus` 或 `westus`。 針對\<mode\>，請指定 `onprem`、`infrastructure`、`createvpn`、`workload`、`security` 或 `all`。

    ```powershell
    .\Deploy-ReferenceArchitecture.ps1 <subscription id> <location> <mode>
    ```   
4. 出現提示時，請登入您的 Azure 帳戶。 部署指令碼可能需要數小時才能完成，視您選取的模式而定。

5. 完成部署之後，請執行指令碼以設定 SQL Server Alwayson 可用性群組。 請參閱[讀我檔案][readme]，以取得詳細資訊。

> [!WARNING]
> 參數檔案會在不同位置中包含硬式編碼的密碼 (`AweS0me@PW`)。 在部署之前，請變更這些值。


## <a name="validate-the-deployment"></a>驗證部署

部署此參考架構之後，下列資源群組會列在您所使用的訂用帳戶底下：

| 資源群組        | 目的                                                                                         |
|-----------------------|-------------------------------------------------------------------------------------------------|
| ra-onprem-sp2016-rg   | 使用 Active Directory 的模擬內部部署網路，與 SharePoint 2016 網路是同盟關係 |
| ra-sp2016-network-rg  | 支援 SharePoint 部署的基礎結構                                                 |
| ra-sp2016-workload-rg | SharePoint 和支援的資源                                                             |

### <a name="validate-access-to-the-sharepoint-site-from-the-on-premises-network"></a>從內部部署網路驗證 SharePoint 網站的存取權

1. 在 [Azure 入口網站][azure-portal]中的 [資源群組] 下，選取 `ra-onprem-sp2016-rg` 資源群組。

2. 在資源清單中，選取名為 `ra-adds-user-vm1` 的虛擬機器資源。 

3. 連線至虛擬機器，如[連線至虛擬機器][connect-to-vm]中所述。 使用者名稱為 `\onpremuser`。

5.  與虛擬機器建立遠端連線後，在虛擬機器中開啟瀏覽器並瀏覽至 `http://portal.contoso.local`。

6.  在 [Windows 安全性] 方塊中，使用 `contoso.local\testuser` 作為使用者名稱來登入 SharePoint 入口網站。

此登入可讓您從內部部署網路使用的 Fabrikam.com 網域通往 SharePoint 入口網站使用的 contoso.local 網域。 當 SharePoint 網站開啟時，您會看到根示範網站。

### <a name="validate-jumpbox-access-to-vms-and-check-configuration-settings"></a>驗證虛擬機器的 jumpbox 存取權及檢查組態設定

1.  在 [Azure 入口網站][azure-portal]中的 [資源群組] 下，選取 `ra-sp2016-network-rg` 資源群組。

2.  在資源清單中，選取名為 `ra-sp2016-jb-vm1` 的虛擬機器資源，這就是 jumpbox。

3. 連線至虛擬機器，如[連線至虛擬機器][connect-to-vm]中所述。 使用者名稱為 `testuser`。

4.  登入 jumpbox 之後，請從 jumpbox 開啟 RDP 工作階段。 連線至 VNet 中的任何其他虛擬機器。 使用者名稱是 `testuser`。 您可以忽略有關遠端電腦安全性憑證的警告。

5.  虛擬機器的遠端連線開啟時，請檢閱設定然後使用「伺服器管理員」等系統管理工具進行變更。

部署的虛擬機器如下表所示。 

| 資源名稱      | 目的                                   | 資源群組        | 虛擬機器名稱                       |
|--------------------|-------------------------------------------|-----------------------|-------------------------------|
| Ra-sp2016-ad-vm1   | Active Directory + DNS                    | Ra-sp2016-network-rg  | Ad1.contoso.local             |
| Ra-sp2016-ad-vm2   | Active Directory + DNS                    | Ra-sp2016-network-rg  | Ad2.contoso.local             |
| Ra-sp2016-fsw-vm1  | SharePoint                                | Ra-sp2016-network-rg  | Fsw1.contoso.local            |
| Ra-sp2016-jb-vm1   | Jumpbox                                   | Ra-sp2016-network-rg  | Jb (使用公用 IP登入) |
| Ra-sp2016-sql-vm1  | SQL Always On - 容錯移轉                  | Ra-sp2016-network-rg  | Sq1.contoso.local             |
| Ra-sp2016-sql-vm2  | SQL Always On - 主要                   | Ra-sp2016-network-rg  | Sq2.contoso.local             |
| Ra-sp2016-app-vm1  | SharePoint 2016 應用程式 MinRole       | Ra-sp2016-workload-rg | App1.contoso.local            |
| Ra-sp2016-app-vm2  | SharePoint 2016 應用程式 MinRole       | Ra-sp2016-workload-rg | App2.contoso.local            |
| Ra-sp2016-dch-vm1  | SharePoint 2016 分散式快取 MinRole | Ra-sp2016-workload-rg | Dch1.contoso.local            |
| Ra-sp2016-dch-vm2  | SharePoint 2016 分散式快取 MinRole | Ra-sp2016-workload-rg | Dch2.contoso.local            |
| Ra-sp2016-srch-vm1 | SharePoint 2016 搜尋 MinRole            | Ra-sp2016-workload-rg | Srch1.contoso.local           |
| Ra-sp2016-srch-vm2 | SharePoint 2016 搜尋 MinRole            | Ra-sp2016-workload-rg | Srch2.contoso.local           |
| Ra-sp2016-wfe-vm1  | SharePoint 2016 Web 前端 MinRole     | Ra-sp2016-workload-rg | Wfe1.contoso.local            |
| Ra-sp2016-wfe-vm2  | SharePoint 2016 Web 前端 MinRole     | Ra-sp2016-workload-rg | Wfe2.contoso.local            |


**_此參考架構的參與者_** &mdash;  Joe Davies、Bob Fox、Neil Hodgkinson、Paul Stork

<!-- links -->

[availability-set]: /azure/virtual-machines/windows/manage-availability
[azure-portal]: https://portal.azure.com
[azure-ps]: /powershell/azure/overview
[azure-pricing]: https://azure.microsoft.com/pricing/calculator/
[bastion-host]: https://en.wikipedia.org/wiki/Bastion_host
[create-availability-group]: https://technet.microsoft.com/library/mt793548(v=office.16).aspx
[connect-to-vm]: /azure/virtual-machines/windows/quick-create-portal#connect-to-virtual-machine
[github]: https://github.com/mspnp/reference-architectures
[hybrid-ra]: ../hybrid-networking/index.md
[hybrid-vpn-ra]: ../hybrid-networking/vpn.md
[load-balancer]: /azure/load-balancer/load-balancer-internal-overview
[managed-disks]: /azure/storage/storage-managed-disks-overview
[minroles]: https://technet.microsoft.com/library/mt346114(v=office.16).aspx
[nsg]: /azure/virtual-network/virtual-networks-nsg
[office-web-apps]: https://support.microsoft.com/help/3199955/office-web-apps-and-office-online-server-supportability-in-azure
[paired-regions]: /azure/best-practices-availability-paired-regions
[readme]: https://github.com/mspnp/reference-architectures/tree/master/sharepoint/sharepoint-2016
[resource-group]: /azure/azure-resource-manager/resource-group-overview
[quotas]: /azure/azure-subscription-service-limits
[sharepoint-accounts]: https://technet.microsoft.com/library/ee662513(v=office.16).aspx
[sharepoint-crawling]: https://technet.microsoft.com/library/dn535606(v=office.16).aspx
[sharepoint-dr]: https://technet.microsoft.com/library/ff628971(v=office.16).aspx
[sharepoint-hybrid]: https://aka.ms/sphybrid
[sharepoint-minrole]: https://technet.microsoft.com/library/mt743705(v=office.16).aspx
[sharepoint-ops]: https://technet.microsoft.com/library/cc262289(v=office.16).aspx
[sharepoint-reqs]: https://technet.microsoft.com/library/cc262485(v=office.16).aspx
[sharepoint-search]: https://technet.microsoft.com/library/dn342836.aspx
[sql-always-on]: /sql/database-engine/availability-groups/windows/always-on-availability-groups-sql-server
[sql-performance]: /virtual-machines/windows/sql/virtual-machines-windows-sql-performance
[sql-server-capacity-planning]: https://technet.microsoft.com/library/cc298801(v=office.16).aspx
[sql-quorum]: https://technet.microsoft.com/library/cc731739(v=ws.11).aspx
[sql-sharepoint-best-practices]: https://technet.microsoft.com/library/hh292622(v=office.16).aspx
[tempdb]: /sql/relational-databases/databases/tempdb-database
[virtual-networks-nsg]: /azure/virtual-network/virtual-networks-nsg
[visio-download]: https://archcenter.blob.core.windows.net/cdn/Sharepoint-2016.vsdx
[vm-sizes-general]: /azure/virtual-machines/windows/sizes-general
[vm-sizes-memory]: /azure/virtual-machines/windows/sizes-memory
[windows-n-tier]: ../virtual-machines-windows/n-tier.md
