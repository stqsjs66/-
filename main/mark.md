# 使用 Sigstore/cosign 实现软件供应链签名与验证 —— 技术实践分析

## 一、背景与动机
- **软件供应链攻击概述**  
近来，软件供应链攻击（Software Supply Chain Attack）已成为一个严重的安全问题，攻击者越来越多地引导目标对准软件的开发和配送渠道，而不是最终产品本身。软件供应链攻击是指攻击者通过渗透或篡改软件开发、构建、发布和分发流程中的某个环节，从而向目标系统传播恶意代码或控制权的一种攻击方式。这种攻击方式近年来呈上升趋势，影响范围广泛且破坏性极大。


-  **SLSA 与 Sigstore**
  
  - **SLSA：Supply-chain Levels for Software Artifacts**  
    SLSA（读作“salsa”）是由 Google 发起的开源框架，用于保障软件供应链中软件构建产物（artifact）的完整性与可追溯性。它定义了一套分级标准，帮助评估和提升软件构建流程的安全性。SLSA 分级模型（v1.0）如下表所示。
  
    | 等级 | 名称              | 说明                                              |
    |------|-------------------|---------------------------------------------------|
    | 1    | 建立可追踪性       | 提供构建产物 provenance（构建信息，如版本、输入来源等） |
    | 2    | 构建自动化         | 使用自动构建系统，避免人为干预                     |
    | 3    | 源码控制 + 可信构建 | 构建系统防篡改，源代码受版本控制，provenance 是可信的 |
    | 4    | 可重建、强身份验证   | 构建过程可重复验证，所有参与者都有强身份认证机制，保障高度完整性 |
  
  - **Sigstore**  
    Sigstore 是一个由 Linux Foundation 支持的开源项目，专为软件供应链设计的自动化签名、验证与发布框架，让开发者可以无需管理密钥地为软件签名。Sigstore Framework 提供了一种统一的方式来签署和验证软件包，确保其来源可靠且未经篡改。它支持多种加密算法，例如 ECDSA、ED25519、RSA 和 DSSE（In-Toto），并且集成了流行的密钥管理系统，如 AWS KMS、Azure Key Vault、HashiCorp Vault 和 Google Cloud Platform KMS。通过这些特性，Sigstore 助力开发者实现对整个软件生命周期的信任和透明度。
  
    | 组件名      |                        作用                        |
    | ----------- | :------------------------------------------------: |
    | **cosign**  |     工具，用于为容器镜像、artifact 签名和验证      |
    | **fulcio**  |   无需预设私钥，自动颁发基于开发者身份的短期证书   |
    | **rekor**   | 透明日志系统，记录所有签名操作，支持公开审计和查证 |
    | **gitsign** |     为 Git 提交签名，集成 GitHub OIDC 身份认证     |


## 二、工具介绍：cosign
- **功能一览**  
  Cosign 是 Sigstore 项目中的核心工具，专注于为容器镜像和各种软件构建产物（artifacts）提供自动化的数字签名与验证功能。其主要功能包括：
  
  - **自动密钥管理**：支持自动生成、存储和使用密钥对，用户无需手动管理复杂的私钥文件，降低操作风险。
  - **身份绑定签名**：结合身份提供者（如 GitHub OIDC、Google 等）实现基于开发者身份的签名，增强签名的可信度。
  - **集成透明日志**：将所有签名记录写入 Rekor 透明日志，实现签名的公开审计和可追溯性，防止签名被篡改或伪造。
  - **支持多种存储后端**：可以将签名附加到容器镜像（如 Docker 镜像）或上传至远程存储，使签名和软件包紧密绑定。
  - **便捷的验证流程**：为用户和自动化系统提供简化的验证接口，确保软件供应链各环节的安全可信。
  
- **与传统签名方式对比**  
  传统的软件签名流程往往依赖手动生成和管理私钥，过程复杂且容易出错，存在以下局限性：
  
  - **私钥管理难度高**：用户需自行保护私钥，密钥泄露风险大，且密钥备份、轮换等运维负担重。
  - **签名与身份脱节**：传统签名仅证明“谁持有私钥”，但难以直接验证签名者的真实身份。
  - **缺乏公开透明审计**：签名记录通常保存在私有系统，无法进行公开验证和防篡改，容易隐藏恶意行为。
  - **流程不够自动化**：签名往往是人工操作，易出错且难以集成到 CI/CD 自动化流水线中。
  
  相比之下，Cosign 通过自动化密钥管理、身份绑定签名以及与透明日志的集成，极大简化了签名流程，提高了签名的安全性和可信度，适合现代云原生和 DevOps 环境下的软件供应链安全需求。

## 三、consign安装与配置过程
- **项目地址**  
  Cosign 是 Sigstore 项目的一部分，源代码托管在 GitHub 上：  
  👉 https://github.com/sigstore/cosign
  安装指令可以从(https://docs.sigstore.dev/cosign/system_config/installation/)中找到
- **安装**  
  使用 Cosign 二进制文件或 rpm/dpkg 包，命令如下：  
 1. 使用二进制文件安装（适用于 Linux）
```bash
curl -O -L "https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64"
sudo mv cosign-linux-amd64 /usr/local/bin/cosign
sudo chmod +x /usr/local/bin/cosign
```
  2. 使用RPM 包安装（适用于 RHEL/CentOS）
```bash
LATEST_VERSION=$(curl https://api.github.com/repos/sigstore/cosign/releases/latest | grep tag_name | cut -d : -f2 | tr -d "v\", ")
curl -O -L "https://github.com/sigstore/cosign/releases/latest/download/cosign-${LATEST_VERSION}-1.x86_64.rpm"
sudo rpm -ivh cosign-${LATEST_VERSION}-1.x86_64.rpm
```
  3. 使用 DPKG 包安装（适用于 Debian/Ubuntu）
```bash
LATEST_VERSION=$(curl https://api.github.com/repos/sigstore/cosign/releases/latest | grep tag_name | cut -d : -f2 | tr -d "v\", ")
curl -O -L "https://github.com/sigstore/cosign/releases/latest/download/cosign_${LATEST_VERSION}_amd64.deb"
sudo dpkg -i cosign_${LATEST_VERSION}_amd64.deb
```
Debian12从二进制文件下载安装consign：

![Debian12从二进制文件安装：](https://github.com/stqsjs66/-/raw/img/img/1.png)
![Debian12从二进制文件安装：](https://github.com/stqsjs66/-/raw/img/img/2.png)

验证安装：

![Debian12从二进制文件安装：](https://github.com/stqsjs66/-/raw/img/img/3.png)

至此consign安装完成。

## 四、签名与验证实操
- **使用无密钥签名**
  可以使用 Cosign 通过 Sigstore 支持的 OIDC (OpenID Connect) 协议进行身份验证，从而使用临时密钥对容器进行签名。
  
  容器无密钥签名的格式如下。
  
  ```bash
  $ cosign sign $IMAGE
  ```
  
  在此之前可以使用 [ttl.sh](https://ttl.sh/) 创建要签名的测试图像， [ttl.sh](https://ttl.sh/) 提供免费、短期（即数小时）、匿名容器镜像托管。要使用 [ttl.sh](https://ttl.sh/) 创建要签名的测试图像(1小时），请运行以下命令：
  
  ```bash
  $ IMAGE_NAME=$(uuidgen)
  $ IMAGE=ttl.sh/$IMAGE_NAME:1h
  $ cosign copy alpine $IMAGE
  ```
  
  debian12 无密钥签名操作，在运行上述操作时可能会由于网络原因遇到以下报错：
  ![无密钥签名操作：](https://github.com/stqsjs66/-/raw/img/img/4.png)
  这是因为国内网络或局域网禁止访问 Docker Hub（或该 IP 被墙），可以手动拉取并上传（绕过 cosign copy),即使用 docker 拉取并推送镜像，然后再使用 cosign 签名：
  
  ```bash
  docker pull alpine
  IMAGE=ttl.sh/$(uuidgen):1h
  docker tag alpine $IMAGE
  docker push $IMAGE
  cosign sign $IMAGE
  ```
  ![无密钥签名操作：](https://github.com/stqsjs66/-/raw/img/img/d3.png)
  ![无密钥签名操作：](https://github.com/stqsjs66/-/raw/img/img/d4.png)
  
  
  
  输入code后，完成无密钥加密
  
  ![无密钥签名操作：](https://github.com/stqsjs66/-/raw/img/img/d5.png)
  
  ![无密钥签名操作：](https://github.com/stqsjs66/-/raw/img/img/d6.png)
  
  
  
- **使用本地密钥对来进行签名与验证**

  如果需要生成本地密钥对，可以运行以下指令：

  ```bash
  cosign generate-key-pair
  ```

  根据提示设置密码，生成的密钥将会存放在本地consign密钥库中。

  ![密钥签名操作：](https://github.com/stqsjs66/-/raw/img/img/k1.png)

  输入以下命令，使用本地密钥对镜像进行签名(注，这里的$IMAGE是需要签名的镜像名)

  ```bash
  $ cosign sign --key cosign.key $IMAGE
  ```

  ![密钥签名操作：](https://github.com/stqsjs66/-/raw/img/img/k2.png)

  使用consign密钥签名成功，下面来验证签名。使用 `cosign verify` 命令的一般验证格式如下：

  ```bash
  cosign verify [--key <key path>|<key url>|<kms uri>] <image uri>
  ```

  ![密钥签名操作：](https://github.com/stqsjs66/-/raw/img/img/k3.png)

  **cosign claims**（签名声明）被成功验证，格式和内容符合预期；

  离线校验了透明日志中是否有对应的签名记录（这里可能是本地缓存或离线验证）；

  签名与提供的公钥 (`--key cosign.pub`) 匹配，签名有效。
  
  更多详细操作见官方[操作文档](https://docs.sigstore.dev/cosign/signing/overview/)。

## 五、优势与不足分析
**-优势**：**安全性与自动化性**

- **安全性**
  - **基于公钥密码学**：cosign 采用公钥加密技术对软件构建产物进行签名，保证签名的真实性和完整性，防止产物在传输或存储过程中被篡改。
  - **透明日志机制**：所有签名操作都会记录在公开透明的日志（Transparency Log）中，任何人都可以验证签名历史，提升了签名的可审计性和不可抵赖性。
  - **自动密钥管理和短期证书**：Sigstore 提供基于身份的短期证书颁发，降低了私钥管理的风险，避免长时间密钥泄露带来的安全隐患。
  - **身份绑定签名**：通过集成 GitHub OIDC 等身份认证机制，确保签名者身份的真实性，增加责任追踪和信任度。

- **自动化**
  - **CI/CD 流水线无缝集成**：cosign 支持在持续集成/持续部署流程中自动执行签名和验证，减少人为操作错误，提高构建安全性。
  - **简化密钥管理**：支持 ephemeral keys（临时密钥）和无密钥签名，降低开发者管理密钥的复杂度，避免密钥泄露和滥用。
  - **丰富的命令行工具和 API**：提供易用的 CLI 工具和编程接口，方便自动化脚本和平台集成，提高开发和运维效率。

**-不足**：**兼容性、可用性挑战**

- **兼容性**
  
  - **对非容器化应用支持有限**：cosign 主要设计用于签名和验证容器镜像，对于传统软件包或非 OCI 标准的构建产物支持不足。
  - **不同平台支持差异**：cosign 在不同操作系统、CPU 架构及环境下的兼容性存在一定差异，可能影响部署和使用体验。
  - **对旧版本容器工具支持有限**：部分旧版容器运行时或镜像仓库可能不完全兼容 cosign 的签名验证机制。
  
- **可用性**
  
  - **依赖网络访问**：签名和验证流程通常依赖 Sigstore 服务器（如 Fulcio、Rekor），网络受限或离线环境下功能受限。
  - **认证流程复杂**：首次使用需要进行 OAuth2 认证和浏览器交互，自动化和无头环境部署存在一定挑战。
  - **学习曲线和流程改造成本**：引入 cosign 需要团队理解新签名机制并改造现有构建流程，对初次使用者存在学习门槛。
  - **签名和验证的性能开销**：签名操作需要访问远端服务并更新日志，验证过程也可能带来一定延迟，影响构建流水线速度。
  
  


## 六、结语
- **实践总结**

本次围绕 Sigstore 工具链（主要包括 cosign）的技术实践，完成了从签名生成到验证流程的全面演示与分析。实践过程中，我们基于实际的容器镜像和 CI/CD 流程，密钥管理（Fulcio）以及签名存储与验证等关键环节，对 Sigstore 体系下的零信任供应链安全理念有了更加直观和深入的理解。

总体而言，cosign 工具提供了较为友好和自动化的使用体验，尤其是在 OIDC 模式下实现无密钥签名的过程中，极大地降低了开发与运维人员使用软件签名机制的门槛。但实践也暴露出其在部署与运行过程中对外部网络资源的较强依赖性，尤其是在国内网络环境下，相关服务的访问稳定性和响应时间对整体流程体验影响较大。

- **建议与未来可能探索方向**

针对实践中遇到的网络依赖性问题，建议 Sigstore 社区未来能进一步增强对本地化部署的支持。例如提供简易的本地化 Fulcio 和 Rekor 部署脚本，或引入镜像源加速、缓存机制，以提升在受限网络环境中的可用性和稳定性。

同时，从企业级应用的角度出发，未来可重点探索以下几个方向：

1. **本地私有化部署方案研究**
    构建基于 Sigstore 工具链的本地私有化解决方案，以适应无法稳定连接公共 OIDC 和透明日志服务的实际场景，特别适用于金融、政企等对数据隔离性要求较高的行业。
2. **与主流 CI/CD 平台的集成优化**
    尝试将 cosign 深度集成到如 GitLab CI、Tekton、Argo CD 等平台中，探索全流程自动化签名与验证的可行策略，形成可复用的流水线模板。
3. **与 SBOM 工具的协同使用研究**
    将 cosign 与 SPDX/CycloneDX 等 SBOM 工具联动，推动签名与软件物料清单的自动化关联，实现更完整的供应链可追溯能力。
4. **对抗签名伪造与攻击的鲁棒性分析**
    深入分析 cosign 生成的签名在面对密钥泄露、Replay 攻击、时钟漂移等安全威胁时的鲁棒性表现，评估其作为可信根机制的安全边界。

通过本次实践，掌握sigstore/cosign 工具的基本使用方法，也意识到其作为开源社区推动的供应链安全解决方案，仍具备广阔的发展与优化空间。未来若能结合本地部署、CI/CD 自动化、SBOM 联动与安全增强机制，相信 Sigstore 在保障软件供应链可信性方面将发挥更大的作用。

## 参考文献
- https://docs.sigstore.dev/
- https://github.com/sigstore/cosign
- 《Supply Chain Security Tooling》PPT
