---
name: cklog-hugo-management
description: >-
  Comprehensive guide, standard conventions, frontmatter schemas, page bundle structures, image naming rules, and publishing workflows for the CK log Hugo blog (CheolKyunYU/ck.log).
---

# CK log Hugo Blog Management Skill

This skill defines the complete operational standards, layout rules, taxonomy configurations, frontmatter schemas, image naming rules, and publishing procedures for the **CK log** blog.

---

## 1. Blog Persona & Identity

* **Blog Name**: `CK notes`
* **Base URL**: `https://cknotes.kr/`
* **Author**: `CK notes` (16-year IT Field Systems Engineer)
* **Core Topics**: Server, Storage, HCI (HPE SimpliVity / VME / VMware), Linux, IT Infrastructure Troubleshooting, Daily Life, Car Maintenance, Travel.

---

## 2. Main Page & Ordering Rules

1. **Pinned Hero Post**:
   * Intro post `content/posts/hello-ck-log.md` has `weight: 1` in frontmatter.
   * Rendered at the very top of the homepage using PaperMod's `first-entry` preview hero style.
2. **Subsequent Post Cards**:
   * All subsequent posts displayed below in standard white card boxes (`post-entry`).
3. **Pagination**:
   * Set to **10 posts per page** (`paginate = 10` in `hugo.toml`).

---

## 3. Categories & Taxonomy Layout Rules

* **Categories**:
  * `Tech`: 기술 및 인프라의 경험 Notes (SimpliVity, VME, Linux, Hardware, Troubleshooting)
  * `일상`: 소개글, 일상, 차량 정비 등
  * `여행`: 여행의 기록
* **Category Layout**: `/categories/` uses a custom vertical tree view (`📂` icons, tree lines, post count badges, indented descriptions).
* **Tag Layout**: `/tags/` maintains PaperMod's default tag pill cloud.

---

## 4. Front Matter & Page Standards

Every post and page MUST include a `description` (~100–120 characters) in its frontmatter:

```yaml
---
title: "[포스트 제목]"
description: "100~120자 내외의 SEO 요약 메타 설명 (포스트 부제목 및 검색 엔진 미리보기에 노출)"
date: YYYY-MM-DDTHH:MM:SS+09:00
draft: false
categories: ["Tech"]
tags: ["HPE", "SimpliVity", "VME", "Troubleshooting"]
---
```

---

## 5. Page Bundle & URL / Image Naming Rules

1. **Hugo Page Bundle Structure**:
   * `content/posts/<bundle_folder>/index.md`
   * `content/posts/<bundle_folder>/images/`
2. **URL & Folder Naming Convention (STRICT)**:
   * **ALL** post folder names and URLs **MUST** use lowercased ASCII kebab-case English only (e.g., `nexo-used-vs-smart-down`, `cisco-mds-snmp-setup-guide`).
   * **NEVER** use Korean characters in folder names or URLs to ensure clean, percent-encoding-free URLs for SEO and social sharing.
3. **Image Naming Rules**:
   * **MUST** use lowercased ASCII filenames without spaces or Korean characters (e.g., `os_network_setup.jpg`, `the_all_new_nexo.jpg`).
   * Avoid spaces or Korean in image filenames to prevent 404 URL encoding errors on Linux web servers (GitHub Pages).
4. **Internal Links Between Posts**:
   * Links must target clean English folder paths (e.g., `../simplivity-00-install-prep/`, `../ollama-01-local-llm-intro/`).

---

## 6. HPE SimpliVity 6.2.0 (HVM) Series Lineup

* `PreStep`: 사전 설치 준비 & 2노드 네트워크 설계 가이드 (`content/posts/simplivity-00-install-prep/index.md`)
* `Step 1`: [관리서버] BaseOS HVM 24.04 & NTP/DNS/NFS 구성 (`content/posts/simplivity-01-baseos-infra-setup/index.md`)
* `Step 2`: [관리서버] VME Manager VM & Arbiter VM 설치 (`content/posts/simplivity-02-vme-mgr-arbiter/index.md`)
* `Step 3`: [SimpliVity 서버] 펌웨어 업데이트 & Initial Setup (`content/posts/simplivity-03-node-initial-setup/index.md`)
* `Step 4`: [클러스터 & OVC 배포] HVM Cluster 생성 & OVC 배포 (`content/posts/simplivity-04-hvm-cluster-ovc-deploy/index.md`)

---

## 7. Verification & Deployment Workflow

1. **Local Build Test**:
   ```powershell
   .\hugo.exe --minify
   ```
2. **Deployment**:
   * Commit and push via **GitHub Desktop**.
   * Triggers GitHub Actions workflow (`.github/workflows/hugo.yml`) for automated deployment.