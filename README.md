code-refactor Skill

本文档总结 Cursor Agent Skill code-refactor 的能力、工作流、优势与使用要点。  

权威 Skill 路径：C:\Users\Zilong Feng\.cursor\skills\code-refactor\  

项目内 .trae/skills/code-refactor/ 为旧版副本，执行时以 Cursor skills 目录为准。

---

1. 定位

code-refactor 是一套行为保持型、可验收的遗留系统重构方法论，用于指导 AI Agent 对既有 Java/Spring 项目进行结构性整理，而非随意改代码。

核心原则：

- 小步：单次单元约 ≤5 文件 / ≤300 行（部分批次有例外）
- 可验证：每单元编译 + 批次门禁
- 可回滚：单元边界清晰，出问题 5 分钟内修不好则回滚
- 有进度：项目根目录维护 REFACTOR_PROGRESS.md

适用场景：历史包袱重（web 包、上帝类、硬编码、错放 DTO）、需不改对外接口契约的渐进式重构。

---

2. 文档体系

  文件                            	用途                   
  SKILL.md                      	主流程：红线、三阶段、九批次、门禁、DoD
  domain-taxonomy.md            	业务域 A/B/C/D 分类与治理规则  
  conventions.md                	目标代码形态：注释、分层、目录、命名、格式
  smells-playbook.md            	坏味道识别 + 《重构清单》模板     
  verification-commands.md      	可执行的 rg 门禁与构建命令      
  safety-checklist.md           	前/中/后期检查与回滚预案        
  batch9-reuse-playbook.md      	批 9：公共复用、复杂拆分、循环/判断优化
  templates/REFACTOR_PROGRESS.md	进度跟踪模板               
  templates/refactor-report.md  	阶段三符合度报告模板           

---

3. 三阶段工作流

阶段一：前期评估（不动业务代码）

  步骤                   	产出                     
  摸底扫描（smells-playbook）	.refactoring/重构清单.md   
  业务域地图（A/B/C/D）       	写入 REFACTOR_PROGRESS.md
  接口基线（URL/方法/入参/出参）   	清单或独立文档                
  安全网评估（测试现状）          	是否补特征测试                
  创建进度文件               	REFACTOR_PROGRESS.md   
  用户确认（硬阻断）            	范围、高危区、优先级；未确认不得进入阶段二  

阶段二：执行（小步循环）

默认策略：按业务域逐个完成，域内按固定批次顺序推进。

推荐域顺序（按类别，非固定包名）：

    D 类（配置）→ C 类（集成/外发）→ 小体量 A 类 → 其余 A 类 → B 类聚合域

每个重构单元循环：

    选取单元 → 改动 → 编译 → 测试 → 接口签名比对 → 批次门禁 → 更新 REFACTOR_PROGRESS → 建议 commit

阶段三：收尾验证

1. 全量构建 + 全量测试 + 接口基线比对  
2. 输出《规范符合度报告》（refactor-report 模板）  
3. 死代码清单逐项问用户后删除  
4. 更新 README / 接口文档  
5. 规范沉淀为 .cursor/rules/  
6. 新踩的坑回写 smells-playbook.md

---

4. 九批次能力一览

域内执行顺序（不可随意混批）：

    批1 → 批2 → 批4 → 批6 → 批5 → 批8 → 批3 → 批7 → 批9

  批次  	名称          	主要内容                                  	风险  
  1   	死代码登记       	注释块、无引用代码——只登记不删                      	无   
  2   	注释规范        	//、行尾注释 → Javadoc                     	极低  
  4   	硬编码外置       	URL/IP/topic/魔法值 → yml + Properties   	低-中 
  6   	目录重组        	web→controller/handler；错放 DTO 回迁；po 收敛	中   
  5   	分层 + Swagger	Service/Impl；逻辑下沉；Swagger 补齐          	中   
  8   	语义命名与格式     	消除 doData/getData；格式统一                	低-中 
  3   	机械重复提取      	复制粘贴级片段 → 方法/小 util                   	低   
  7   	拆上帝类        	400+ 行类 Extract Class                 	中-高 
  9   	公共复用与逻辑优化   	跨类提取、长方法拆分、循环/判断简化（域内最后）              	低-中 

批 3 与批 9 的分工

      	批 3                    	批 9                           
  时机  	批 8 之后                 	批 3、7 之后                      
  对象  	机械复制粘贴的短片段             	跨类公共逻辑、长方法、深嵌套控制流             
  示例  	桩号 replace(" ", "+") 重复	StatisWebServiceImpl 内多段相似统计逻辑

进度符号（REFACTOR_PROGRESS.md）

  符号  	含义                
  ✅   	本域该批次全部完成且门禁通过    
  🔄  	已开工、有成果，但仍有遗留或未跑门禁
  ⬜   	未开始               
  ⏭   	跳过（须注明原因）         

---

5. 业务域分类（A/B/C/D）

  类别  	名称    	识别信号            	本项目示例             
  A   	核心域   	独立实体/表；单一业务能力   	device、video、b5、e1
  B   	聚合/编排域	跨多域拼装；大屏/统计     	statis            
  C   	集成域   	对外 HTTP/MQ；第三方对接	send              
  D   	基础设施  	config、util、框架扩展	config、util       

B 类特别注意：{聚合域}/dto 只放跨域拼装对象；单一来源 DTO 必须迁回对应 A 类域。

---

6. 红线规则（8 条）

1. 禁止改变对外接口契约（URL、HTTP 方法、入参名、出参结构）  
2. 禁止重构与修 bug 混合——bug 单独登记、单独提交  
3. 禁止大爆炸重写——超 5 文件/300 行须拆单元  
4. 禁止未编译验证就进入下一单元  
5. 死代码不立即删除——登记，阶段三统一决策  
6. 删除/移动前全局搜引用（含 XML、yml、反射字符串）  
7. 高危区默认不碰（分库分表、MQ topic、推送 JSON 结构、密钥）  
8. 禁止跳过批次门禁

---

7. 执行纪律（防混乱）

以下规则用于避免「多批混做、进度表看不懂」：

1. 一单元一批次：单元标题格式 {域名}-批{N} 或 {域名}-批9.1  
2. 开批前列清单：在进度文件填写「本批待办文件表」  
3. 🔄 与 ✅ 分清：门禁未过只能标 🔄，在「批次明细」写清已完成/未完成  
4. 跳批须记录：在「当前阻塞」注明原因  
5. 批 9 最后做：结构、命名、拆类完成后再做复用与控制流优化  

---

8. 批次门禁（摘要）

门禁命令详见 Skill 内 verification-commands.md。常用检查：

  批次  	通过标准（摘要）                                
  2   	声明行无 // 说明注释；无行尾字段注释                    
  4   	业务 Java 无 URL/IP 字面量：rg "https?://\|\"10\.\d+\." ...
  5   	Controller 不注 Mapper、不分页；对外接口有 Swagger  
  6   	无 Controller 留在 web 包；po 只减不增           
  8   	无 do/handle/getData 等模糊方法名              
  7   	目标上帝类行数下降                               
  9   	超长方法/深嵌套较基线下降；行为不变                      

构建命令（本项目）：

    mvn -pl server-road/road_control -am compile -q

---

9. 主要优势

  优势            	说明                                      
  可预期           	批次 + 门禁 + DoD，「做完」可检查                   
  项目无关          	阶段一动态划域，不写死包名                           
  贴合 Spring 遗留痛点	web 包、PageHelper 在 Controller、枚举 IP、错放 DTO
  行为保持          	适合线上系统；接口契约红线                           
  可恢复协作         	REFACTOR_PROGRESS.md 跨会话接续              
  风险分层          	低危批先跑，拆类/分层后置                           
  收尾有质量关        	批 9 补公共复用与控制流，非只「能编译」                   

---

10. 局限与前提

  局限         	说明                             
  依赖 Agent 自律	Skill 是文档约束，混批仍需人工看进度表         
  门禁为启发式     	rg 抓不到圈复杂度、语义级重复               
  测试薄弱时      	「行为不变」主要靠编译 + 接口基线，批 5/7 仍有风险  
  批次顺序有学习成本  	4 在 5 前、8 在 3 前——先归位/外置，再分层/重命名

---

11. 快速上手（给执行者）

1. 阅读 Skill 内 SKILL.md 红线与批次顺序  
2. 确认 REFACTOR_PROGRESS.md 与《重构清单》已存在  
3. 与用户确认：范围、域顺序、是否动高危区  
4. 选定一个域 + 一个批次，填写「本批待办文件表」  
5. 改动 → 编译 → 跑该批门禁 → 更新进度 → 建议 commit  
6. 本批 ✅ 后再开下一批；本域全部批次完成后再换域  

触发 Skill 的关键词：重构、代码清理、消除硬编码、目录重组、拆分服务、公共代码提取、循环优化等。

---


