---
title: 選擇 Azure 計算服務的準則
description: 比較數個軸間的 Azure 計算服務
author: MikeWasson
layout: LandingPage
ms.date: 04/21/2018
ms.openlocfilehash: ff90ec41c56ae0ecb81bc82128f02fd06d02cb32
ms.sourcegitcommit: d702b4d27e96e7a5a248dc4f2f0e25cf6e82c134
ms.translationtype: HT
ms.contentlocale: zh-TW
ms.lasthandoff: 04/23/2018
---
# <a name="criteria-for-choosing-an-azure-compute-service"></a>選擇 Azure 計算服務的準則

*計算*一詞是指您的應用程式執行所在運算資源的裝載模型。 下表比較數個軸間的 Azure 計算服務。 為您的應用程式選取計算選項時，請參考這些表格。

## <a name="hosting-model"></a>裝載模型

| 準則 | 虛擬機器 | App Service 方案 | Service Fabric | Azure Functions | Azure Container Service | Container Instances | Azure Batch |
|----------|-----------------|-------------|----------------|-----------------|-------------------------|----------------|-------------|
| 應用程式組合 | 無從驗證 | [應用程式] | 服務、來賓可執行檔、容器 | Functions | 容器 | 容器 | Scheduled jobs  |
| 密度 | 無從驗證 | 每個執行個體多個應用程式，透過應用程式方案 | 每個 VM 多個服務 | 沒有專用執行個體 <a href="#note1"><sup>1</sup></a> | 每個 VM 多個容器 |沒有專用執行個體 | 每個 VM 多個應用程式 |
| 最小節點數目 | 1 <a href="#note2"><sup>2</sup></a>  | 1 | 5 <a href="#note3"><sup>3</sup></a> | 沒有專用節點 <a href="#note1"><sup>1</sup></a> | 3 | 沒有專用節點 | 1 <a href="#note4"><sup>4</sup></a> |
| 狀態管理 | 無狀態或可設定狀態 | 無狀態 | 無狀態或可設定狀態 | 無狀態 | 無狀態或可設定狀態 | 無狀態 | 無狀態 |
| Web 裝載 | 無從驗證 | 內建 | 無從驗證 | 不適用 | 無從驗證 | 無從驗證 | 否 |
| 作業系統 | Windows、Linux | Windows、Linux  | Windows、Linux | 不適用 | Windows (預覽版)、Linux | Windows、Linux | Windows、Linux |
| 可以部署到專用的 VNet 嗎？ | 支援 | 支援 <a href="#note5"><sup>5</sup></a> | 支援 | 不支援 | 支援 | 不支援 | 支援 |
| 混合式連線 | 支援 | 支援 <a href="#note1"><sup>6</sup></a>  | 支援 | 不支援 | 支援 | 不支援 | 支援 |

注意

1. <span id="note1">如果是使用使用情況方案。 如果是使用 App Service 方案，則函式會在配置給您的 App Service 方案的 VM 上執行。 請參閱[選擇正確的 Azure Functions 服務方案][function-plans]。</a>
2. <span id="note2">具有兩個或更多個執行個體的較高 SLA。</a>
3. <span id="note3">適用生產環境。</a>
4. <span id="note4">作業完成後可相應減少為零。</a>
5. <span id="note5">需要 App Service Environment (ASE)。</a>
6. <span id="note7">需要 ASE 或 BizTalk 混合式連線</a>

## <a name="devops"></a>DevOps

| 準則 | 虛擬機器 | App Service 方案 | Service Fabric | Azure Functions | Azure Container Service | Container Instances | Azure Batch |
|----------|-----------------|-------------|----------------|-----------------|-------------------------|----------------|-------------|
| 本機偵錯 | 無從驗證 | IIS Express、其他<a href="#note1b"><sup>1</sup></a> | 本機節點叢集 | Azure Functions CLI | 本機容器執行階段 | 本機容器執行階段 | 不支援 |
| 程式設計模型 | 無從驗證 | Web 應用程式、背景工作的 WebJobs | 來賓可執行檔、服務模型、執行者模型、容器 | 具有觸發程序的函式 | 無從驗證 | 無從驗證 | 命令列應用程式 |
| 應用程式更新 | 沒有內建支援 | 部署位置 | 輪流升級 (每個服務) | 沒有內建支援 | 取決於協調器。 大部分支援輪流更新 | 更新容器映像 | 不適用 |

注意

1. <span id="note1b">選項包括 IIS Express for ASP.NET 或 node.js (iisnode)；PHP Web 伺服器；Azure Toolkit for IntelliJ、Azure Toolkit for Eclipse。 App Service 也支援已部署 Web 應用程式的遠端偵錯。</a>
2. <span id="note2b">請參閱 [Resource Manager 提供者、區域、API 版本及結構描述][resource-manager-supported-services]。 


## <a name="scalability"></a>延展性

| 準則 | 虛擬機器 | App Service 方案 | Service Fabric | Azure Functions | Azure Container Service | Container Instances | Azure Batch |
|----------|-----------------|-------------|----------------|-----------------|-------------------------|----------------|-------------|
| 自動調整 | VM 擴展集 | 內建服務 | VM 擴展集 | 內建服務 | 不支援 | 不支援 | N/A |
| 負載平衡器 | Azure Load Balancer | 整合式 | Azure Load Balancer | 整合式 | Azure Load Balancer |  沒有內建支援 | Azure Load Balancer |
| 調整限制 | 平台映像：每個 VMSS 1000 個節點，自訂映像：每個 VMSS 100 個節點 | 20 個執行個體，App Service 環境 50 個 | 每個 VMSS 100 個節點 | 無限 <a href="#note1c"><sup>1</sup></a> | 100 |每一訂用帳戶 20 個容器群組 <a href="#note2c"><sup>2</sup></a> | 預設為 20 個核心限制。 請連絡客戶服務來增加。 |

注意

1. <span id="note1c">如果是使用使用情況方案。 如果是使用 App Service 方案，則適用 App Service 調整限制。 請參閱[選擇正確的 Azure Functions 服務方案][function-plans]。</a>
2. <span id="note2c">請參閱 [Azure Container Instances 的配額和區域可用性](/azure/container-instances/container-instances-quotas)。</a>


## <a name="availability"></a>可用性

| 準則 | 虛擬機器 | App Service 方案 | Service Fabric | Azure Functions | Azure Container Service | Container Instances | Azure Batch |
|----------|-----------------|-------------|----------------|-----------------|-------------------------|----------------|-------------|
| SLA | [虛擬機器的 SLA][sla-vm] | [App Service 的 SLA][sla-app-service] | [Service Fabric 的 SLA][sla-sf] | [Functions 的 SLA][sla-functions] | [Azure Container Service 的 SLA][sla-acs] | [容器執行個體的 SLA](https://azure.microsoft.com/support/legal/sla/container-instances/) | [Azure Batch 的 SLA][sla-batch] |
| 多區域容錯移轉 | 流量管理員 | 流量管理員 | 流量管理員、多區域叢集 | 不支援  | 流量管理員 | 不支援 | 不支援 |

## <a name="other"></a>其他

| 準則 | 虛擬機器 | App Service 方案 | Service Fabric | Azure Functions | Azure Container Service | Container Instances | Azure Batch |
|----------|-----------------|-------------|----------------|-----------------|-------------------------|----------------|-------------|
| SSL | 已在 VM 中設定 | 支援 | 支援  | 支援 | 已在 VM 中設定 | 不支援 | 支援 |
| 成本 | [Windows][cost-windows-vm]、[Linux][cost-linux-vm] | [App Service 價格][cost-app-service] | [Service Fabric 價格][cost-service-fabric] | [Azure Functions 價格][cost-functions] | [Azure Container Service 價格][cost-acs] | [容器執行個體價格](https://azure.microsoft.com/pricing/details/container-instances/) | [Azure Batch 價格][cost-batch]
| 合適的架構樣式 | 多層式架構、Big Compute (HPC) | Web 佇列背景工作 | 微服務、事件驅動架構 (EDA) | 微服務、EDA | 微服務、EDA | 微服務、工作自動化、批次作業  | Big Compute |

[cost-linux-vm]: https://azure.microsoft.com/pricing/details/virtual-machines/linux/
[cost-windows-vm]: https://azure.microsoft.com/pricing/details/virtual-machines/windows/
[cost-app-service]: https://azure.microsoft.com/pricing/details/app-service/
[cost-service-fabric]: https://azure.microsoft.com/pricing/details/service-fabric/
[cost-functions]: https://azure.microsoft.com/pricing/details/functions/
[cost-acs]: https://azure.microsoft.com/pricing/details/container-service/
[cost-batch]: https://azure.microsoft.com/pricing/details/batch/

[function-plans]: /azure/azure-functions/functions-scale
[sla-acs]: https://azure.microsoft.com/support/legal/sla/container-service/
[sla-app-service]: https://azure.microsoft.com/support/legal/sla/app-service/
[sla-batch]: https://azure.microsoft.com/support/legal/sla/batch/
[sla-functions]: https://azure.microsoft.com/support/legal/sla/functions/
[sla-sf]: https://azure.microsoft.com/support/legal/sla/service-fabric/
[sla-vm]: https://azure.microsoft.com/support/legal/sla/virtual-machines/

[resource-manager-supported-services]: /azure/azure-resource-manager/resource-manager-supported-services