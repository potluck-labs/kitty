# GDPR duties for a small app storing financial data

Research for [#6](https://github.com/potluck-labs/kitty/issues/6), part of the map [#2](https://github.com/potluck-labs/kitty/issues/2). Researched on 2026-10-28.

This is engineering research, not legal advice. It reads the GDPR text, EDPB guidance, one national authority (CNIL), and Telegram's own terms, and turns them into design constraints. The map already lists legal review of the privacy notice as out of scope. The open questions at the end are the points a lawyer should check before launch.

## Question

What does GDPR require of a small, free, EU-run app that stores personal expense and account data for any User who signs up through a Telegram bot? Cover the privacy notice, lawful basis, access and export, erasure and how it interacts with shared Group Transactions, retention, processors such as the VPS host and Telegram, and hosting location.

## Short answer

GDPR applies to Kitty in full, even though Kitty is free and its Users record household spending. The duties that change the design are these:

1. Use contract (Art. 6(1)(b)) as the lawful basis for the core service. Anything beyond running the service, such as usage analytics, needs its own basis.
2. Show a privacy notice before the first Transaction is stored, from inside the bot, and register it in @BotFather.
3. Build a self-service export of everything about a User, in JSON or CSV, that includes shared Group Transactions. Answer any rights request within one month.
4. Build account deletion that removes the User and their personal data, and replaces their identity in shared Groups with a placeholder so other Members keep correct balances.
5. Set a retention period for every table, logs, and backups, and write them down. Delete or anonymise on a schedule.
6. Sign the VPS host's Article 28 data processing agreement. Host in the EU or EEA.
7. Encrypt data at rest with the key stored apart from the data. Telegram's bot terms require this.
8. Keep a written record of processing and a breach log, and be able to notify the authority within 72 hours.

Telegram is not Kitty's processor. It is a separate controller, and whether sending bot replies through it counts as a transfer outside the EEA is an open question.

## 1. Scope: GDPR applies, and the two of you are the controller

- Art. 2(2)(c) exempts processing "by a natural person in the course of a purely personal or household activity". Users tracking their own and their friends' spending fall under this. Kitty does not. Recital 18 says the Regulation "applies to controllers or processors which provide the means for processing personal data for such personal or household activities". [GDPR Art. 2, Recital 18]
- Art. 3(1) applies the GDPR to processing "in the context of the activities of an establishment of a controller or a processor in the Union, regardless of whether the processing takes place in the Union or not". Kitty is run from the EU, so this holds wherever the server is. [GDPR Art. 3]
- Being free does not change anything. No article exempts non-commercial services that are open to the public.
- A controller is whoever "determines the purposes and means of the processing" (Art. 4(7)). Without a company, that is the two of you as natural persons. Two people who jointly decide are joint controllers under Art. 26. They must agree in a "transparent manner" who handles which duty, in particular rights requests and the privacy notice, and make "the essence of the arrangement" available to data subjects. A Member can exercise rights against either of you (Art. 26(3)). If you later form a company, the company becomes the controller and the notice changes. [GDPR Art. 4(7), Art. 26]

Design consequence: one short joint-controller arrangement in the repo, and a single contact address in the notice.

## 2. Lawful basis

Art. 6(1) lists six bases. Two fit Kitty:

- Art. 6(1)(b), "necessary for the performance of a contract to which the data subject is party". The free terms of use are the contract. Recording Transactions, computing Shared balances and Loan balances, and sending replies in the bot are the service itself.
- Art. 6(1)(f), legitimate interests, for things that are useful but not the service.

The EDPB reads "necessary" narrowly. Processing must be "objectively necessary" for the service, and "if there are realistic, less intrusive alternatives, the processing is not 'necessary'" (Guidelines 2/2019, para. 25). It also says Art. 6(1)(b) is not generally an appropriate basis for "improving a service or developing new functions" (para. 49), and that fraud prevention usually needs Art. 6(1)(f) instead (para. 50). [EDPB Guidelines 2/2019]

Consent (Art. 6(1)(a)) is a poor fit for the core service. It must be withdrawable at any time (Art. 13(2)(c)), and withdrawal would remove data other Members depend on. Consent would also trigger Art. 8, which sets a minimum age of 16 (13 to 16 by national law) for consent-based information society services. With contract as the basis, Art. 8 does not apply, and national contract law decides whether minors can agree to the terms (Art. 8(3)). [GDPR Art. 6, 8, 13]

One conflict to note: Telegram's Bot Developer Terms §4.3 say developers may use "data submitted directly and voluntarily" by users only if the users "give their individual, explicit, active and revocable consent". This is a contractual term with Telegram, not a GDPR requirement. A simple fix is an explicit "I agree" step at sign-up that accepts the terms and acknowledges the notice, while the GDPR basis stays contract. [Telegram Bot Developer Terms §4.3]

Design consequence: list each purpose with its basis. Core ledger: 6(1)(b). Operational logs and abuse prevention: 6(1)(f). No analytics in the MVP unless you add a basis and a notice line for it.

## 3. Privacy notice contents and delivery

Art. 13 applies because Kitty collects data from the User. It must be given "at the time when personal data are obtained". Required items:

- Art. 13(1): (a) identity and contact details of the controller, (b) DPO contact if any, (c) purposes and legal basis for each, (d) the legitimate interests where 6(1)(f) is used, (e) recipients or categories of recipients, (f) any transfer outside the EEA and the safeguard used.
- Art. 13(2): (a) retention period or the criteria for it, (b) the rights of access, rectification, erasure, restriction, objection, and portability, (c) the right to withdraw consent if consent is used, (d) the right to complain to a supervisory authority, (e) whether providing data is a contractual requirement and what happens if the User does not, (f) any automated decision-making. Kitty has none.

[GDPR Art. 13]

Art. 14 covers data not obtained from the data subject. In Kitty, a Member records an Expense whose Split names another Member. That data comes from someone else. Art. 14(5)(a) lifts the duty "where and insofar as the data subject already has the information", which holds if every Member sees the notice when joining through an Invite code. The notice should say that other Members of a Group can record Transactions that name you. [GDPR Art. 14]

If Kitty uses Art. 6(1)(f), Art. 21(4) requires the right to object to be "explicitly brought to the attention of the data subject" at the latest at the first communication, "clearly and separately from any other information". [GDPR Art. 21(4)]

Delivery in a bot:

- WP29 transparency guidelines (endorsed by the EDPB) mention a "chatbot interface" as an accepted way to provide information, and say in-app information should be "never more than 'two taps away'". A `/privacy` command and a link in the first message before sign-up meet this. [WP260 rev.01, para. 11]
- Telegram's Bot Developer Terms §4 require every bot to have a privacy policy "easily accessible to their users". Bots whose practices differ from Telegram's Standard Bot Privacy Policy "must set up a Privacy Policy in @BotFather". Kitty stores financial data, so it should register its own. [Telegram Bot Developer Terms §4]

Design consequence: the notice is a versioned file in the repo, served at a public URL, linked from @BotFather, and shown before the first Transaction is stored. Store which notice version each User accepted. The notice must list Telegram as a recipient, and the VPS host as a processor.

## 4. Right of access and data export

Access (Art. 15):

- The User gets confirmation, a copy of the data, and the information in Art. 15(1)(a) to (h). For electronic requests the copy comes "in a commonly used electronic form" (Art. 15(3)).
- Art. 15(4): the copy "shall not adversely affect the rights and freedoms of others". The EDPB says this "should not result in refusing the data subject's request altogether". At most, parts that harm others are left out or made illegible. The controller "must be able to demonstrate" a concrete harm, and a general concern is not enough. [EDPB Guidelines 01/2022, paras. 168 to 173]
- The EDPB names a self-service "Download your personal data" option as an accepted way to answer access requests, with two limits: the tool "should never limit the scope of personal data received", and the controller must also handle requests sent through other channels, such as email. [EDPB Guidelines 01/2022, paras. 137 to 138]

Portability (Art. 20):

- Applies because the basis is contract and processing is automated. The User gets data they "provided" in "a structured, commonly used and machine-readable format".
- WP29 guidance (endorsed by the EDPB) counts data the User actively entered and data observed from their use of the service as "provided". It excludes inferred or derived data, such as computed balances, though including them does no harm. It names "XML, JSON, CSV" as suitable formats. [WP242 rev.01]
- On records that involve other people, WP242 says controllers "should not take an overly restrictive interpretation". Its bank example says an account history with other people's transfers still belongs in the export. A Kitty Group's Transactions are the same case. The Member already sees these Transactions in the Group, so leaving them out is hard to justify. [WP242 rev.01, first and third conditions]

Deadlines and cost (Art. 12):

- Answer "without undue delay and in any event within one month of receipt". You can extend by two months for complex or numerous requests if you tell the User within the first month. [Art. 12(3)]
- Free of charge, except for manifestly unfounded or excessive requests (Art. 12(5)).
- If you have reasonable doubts about who is asking, you may ask for more information (Art. 12(6)). A request from the linked Channel identity is a reasonable check.

Design consequence: an `/export` command that returns one JSON file (or a ZIP of CSVs) with the User, linked Channel identities, Accounts, all Transactions they recorded or appear in, Group memberships, and the accepted notice version. Build it on a backend endpoint, not in the Telegram adapter, so it works for later Channels (ADR 0001).

## 5. Right to erasure and shared Group Transactions

What the law says:

- Art. 17(1) lists six grounds for erasure. For Kitty the likely ones are (a) data "no longer necessary", which applies once the User leaves, and (c) an objection under Art. 21(1) to processing based on legitimate interests where there are "no overriding legitimate grounds".
- Art. 17(3) lists exceptions. The relevant one is (e), "the establishment, exercise or defence of legal claims". It is narrow and applies to specific claims, not to a general wish to keep balances tidy.
- EDPB Guidelines 2/2019 para. 41: when a contract ends, processing under 6(1)(b) "will no longer be necessary", and "it is generally unfair to swap to a new legal basis when the original basis ceases to exist". Para. 43: if a controller needs to keep records after the contract, it must "identify a legal basis for this at the outset" and say "from the start" how long it keeps them.
- Recital 26: pseudonymised data that "could be attributed to a natural person by the use of additional information" is still personal data. Only data that no longer relates to an identifiable person is outside the GDPR.

[GDPR Art. 17, 21, Recital 26; EDPB Guidelines 2/2019 paras. 41 to 43]

The conflict: an Expense paid by Member A with a Share for Member B is personal data of both. If A deletes their account and Kitty deletes every Transaction that names A, B's Shared balance changes and B's history has gaps. The other Members still need the record.

Options:

| Option | What happens on deletion | GDPR position | Cost |
| --- | --- | --- | --- |
| A. Hard delete everything | Delete the User and every Transaction that involves them. | Clean erasure. Breaks other Members' balances and history. | Simple code, poor product. |
| B. Replace identity in shared Groups | Delete the User, Channel identities, Accounts, and all personal data outside shared Groups. In shared Groups, re-point their Member row to a placeholder such as "Former member", drop Account names, keep amounts, dates, Categories. | The User's identity goes. The remaining rows are probably still personal data, because other Members remember who the placeholder was (Recital 26). The basis for keeping them is the remaining Members' legitimate interest in a correct ledger (6(1)(f)), which must be named in the notice from day one (EDPB 2/2019 para. 43). | Moderate. Needs a placeholder Member and careful foreign keys. |
| C. Keep everything until balances are zero | Refuse deletion while the User has non-zero balances. | Weak. GDPR has no "settle first" exception, and Art. 17(3)(e) covers legal claims, not routine balances. | Blocks a right, risky. |

Leaning: option B, with two additions. First, warn the User about open balances before deletion and offer a final Settlement, without blocking. Second, state in the notice from the start that shared Group Transactions stay in the Group under a placeholder after you leave. A Group with one Member (a personal ledger) is deleted in full. This is a recommendation for the two of you to decide, and a point for legal review.

Related duties:

- Art. 19: tell each recipient about erasure "unless this proves impossible or involves disproportionate effort". Telegram chat history with past bot replies stays in Telegram and Kitty cannot delete it from Members' chats. The notice should say so.
- Telegram Bot Developer Terms §4.2(a) independently require deleting user data on the user's request.
- Leaving a Group is a separate case from deleting the account. It is already listed under "Invite code lifecycle" on the map. The same placeholder mechanism can serve both.

## 6. Data retention

- Art. 5(1)(e), storage limitation: keep data in identifiable form "for no longer than is necessary for the purposes".
- Recital 39: "time limits should be established by the controller for erasure or for a periodic review".
- Art. 13(2)(a): the notice must state the period or the criteria.
- The GDPR sets no fixed number of days. The controller picks and justifies them.

CNIL's developer guide (France's authority) describes three stages: active database, restricted intermediate archive, then deletion or anonymisation. It says not to leave data in active tables just by flagging it as archived, and to reuse the erasure code path for scheduled purges. Its example periods include 6 months for log data. [CNIL Sheet n°14]

Telegram Bot Developer Terms §4.2(b) and (c) require deletion when retention "becomes unnecessary to operate your TPA", and deletion of all data when the bot stops operating. [Telegram Bot Developer Terms §4.2]

I found no primary source with a rule for backups. A defensible design is a fixed backup rotation, such as 30 days, written in the notice. A deleted User then disappears from backups once the rotation passes, and a restore re-runs pending deletions.

Example policy to decide on:

| Data | Proposed retention | Reason |
| --- | --- | --- |
| User, Accounts, Transactions | While the account exists | Needed for the service. |
| Inactive accounts | Warn after 18 months with no activity, delete after 24 | Storage limitation. The numbers are a proposal, not from a source. |
| Shared Group Transactions after a Member deletes | Kept under placeholder until the Group is deleted | Remaining Members' legitimate interest. |
| Application logs | 6 months at most, without Transaction contents | CNIL example for logs. |
| Backups | Fixed rotation, such as 30 days | No source. Keep it short and stated. |

## 7. Processors, recipients, and Telegram

VPS host:

- The EDPB treats a hosting service that only stores data as a processor, even for "take it or leave it" standard contracts. The customer remains controller and "a data processing agreement according to Article 28 must be concluded". [EDPB Guidelines 07/2020, examples on cloud storage and hosting services]
- Art. 28(3) lists what the agreement must contain: processing only on documented instructions, confidentiality, Art. 32 security, sub-processor rules, help with rights requests and breaches, deletion or return at the end, and audit information. Most EU VPS providers publish a standard agreement that customers accept in their console. Check that it covers these points and lists sub-processors. [GDPR Art. 28]
- Art. 28(1): use only processors "providing sufficient guarantees".

Telegram:

- Telegram's privacy policy names Telegram Messenger Inc. as "the data controller responsible for your personal data", and says it is located outside the EEA with EDPO as its EU representative. It stores data of users who signed up from the EEA "in data centers in the Netherlands". [Telegram Privacy Policy §1.4, §4.1]
- The same policy says bots are run by "third-party bot developers" and receive the user's messages, "screen name, username and profile picture(s)", and interface language. [Telegram Privacy Policy §6]
- Kitty cannot give Telegram instructions or sign an Art. 28 agreement with it. Telegram decides its own purposes and means. My reading is that Telegram is an independent controller and a recipient, not Kitty's processor. List it as a recipient in the notice.
- Telegram's Bot Developer Terms add duties of their own: a privacy policy (§4), deletion on request and when no longer needed (§4.2), no collection beyond what the service needs (§4.3), data "encrypted at rest and stored separately from its encryption key" (§4.4(a)), and breach alerts to users "in accordance with applicable laws" (§4.4(b)). Breaking them can get the bot removed.

Other recipients to list in the notice: the other Members of each Group, who see Transactions that name you.

## 8. Hosting location and transfers

- The GDPR has no data localisation rule. What it controls is transfers to third countries (Chapter V, Art. 44).
- EDPB Guidelines 05/2021 define a transfer by three cumulative criteria: an exporter subject to the GDPR, which discloses data to another controller or processor (the importer), which is in a third country. [EDPB Guidelines 05/2021, section 2]
- A VPS in an EU or EEA data centre, run by an EU company, meets none of the third-country criteria, so Chapter V does not apply. This is the simplest choice.
- A non-EEA host needs an adequacy decision (Art. 45) or safeguards such as standard contractual clauses (Art. 46). For the US, the EU-US Data Privacy Framework (Commission Implementing Decision (EU) 2023/1795 of 10 July 2023) covers only US organisations certified under it.
- Telegram: when the bot sends a reply that names Members and amounts, Kitty arguably discloses data to Telegram Messenger Inc., a controller outside the EEA. That would meet all three criteria. Art. 49(1)(b) allows transfers "necessary for the performance of a contract between the data subject and the controller", which fits a User's own data. It fits less well for other Members' data in a Group message. Members also chose Telegram themselves and already send it their messages. This is unresolved. See open questions.

Design consequence: pick an EU-based VPS provider with an EU data centre. Keep personal data in bot replies to what the reply needs. Keep error tracking, email, and backup storage in the EU too, or each one becomes a processor with a transfer.

## 9. Security and breaches

- Art. 32(1) asks for security "appropriate to the risk", naming "pseudonymisation and encryption", resilience, timely restore, and regular testing.
- Telegram §4.4(a) makes encryption at rest with a separately stored key a hard requirement for the bot.
- Art. 33(1): notify the supervisory authority "not later than 72 hours after having become aware" of a breach, unless it is "unlikely to result in a risk". Art. 33(5): document every breach, including ones you do not report.
- Art. 34(1): tell affected Users if the breach is "likely to result in a high risk". Art. 34(3)(a) lifts this if the data was encrypted so it is "unintelligible to any person who is not authorised to access it". Encryption at rest therefore also reduces notification duties.
- EDPB Guidelines 9/2022 on breach notification give worked examples.

Design consequence: disk or column encryption with keys outside the database host's data volume, TLS between the Channel adapter and the backend, a tested restore, a `docs/` breach log template, and a one-page runbook with the national authority's notification form.

## 10. Paperwork duties

- Records of processing (Art. 30): Art. 30(5) exempts organisations with fewer than 250 people unless the processing "is not occasional", among other conditions. WP29 says a processing activity is "occasional" only if it "is not carried out regularly, and occurs outside the regular course of business". Kitty's core processing is regular, so you need a record. It is a short written table: purposes, categories of data and data subjects, recipients, transfers, erasure time limits, and a security summary (Art. 30(1)). [GDPR Art. 30; WP29 position paper on Art. 30(5)]
- Data protection impact assessment (Art. 35): required where processing is "likely to result in a high risk". WP29 lists nine criteria and says "in most cases" meeting two calls for a DPIA. Kitty meets "sensitive data or data of a highly personal nature", which names "financial data that might be used for payment fraud". It does not meet "large scale" at MVP size. One criterion alone usually does not require a DPIA, but the national authority's Art. 35(4) list may say otherwise. A short DPIA also covers most of the content for the record of processing and the notice. [GDPR Art. 35; WP248 rev.01]
- Data protection officer (Art. 37): required only for public bodies, large-scale regular monitoring, or large-scale special categories. Kitty needs none.
- EU representative (Art. 27): only for controllers outside the EU. Not needed.

## 11. Special categories risk

Art. 9 prohibits processing data revealing health, religion, political opinion, and other listed categories unless an Art. 9(2) exception applies. The CJEU held in C-252/21 (Meta v Bundeskartellamt, 4 July 2023) that data from purchases and visits counts as special category data "where that data processing allows information falling within one of those categories to be revealed", whether about the user or another person. [GDPR Art. 9; CJEU C-252/21, ruling point 2]

Kitty does not profile, but a Category such as "Medical", or Merchants such as a pharmacy or a place of worship, linked to a named Member could reveal health or religion. Art. 9(2)(a) explicit consent would be the likely exception if this counts. Data minimisation helps: a fixed Category list without health or religion labels, free-text Merchant left optional, and no inference.

## Design checklist

| Duty | Source | Design consequence |
| --- | --- | --- |
| Lawful basis per purpose | Art. 6(1)(b), (f); EDPB 2/2019 | Purpose table in the notice. No analytics in MVP. |
| Notice before collection | Art. 13, 14, 21(4); WP260; Telegram §4 | Versioned notice, `/privacy`, @BotFather link, acceptance step, stored version per User. |
| Access and portability | Art. 15, 20; EDPB 01/2022; WP242 | Backend export endpoint, JSON or CSV, includes shared Group Transactions. |
| One-month response | Art. 12(3) | Contact address, a request log, a test of export and deletion in CI. |
| Erasure | Art. 17, 19; Telegram §4.2 | Account deletion job, placeholder Member in shared Groups, full delete of personal ledgers. |
| Retention | Art. 5(1)(e), 13(2)(a); CNIL Sheet 14 | Retention table, scheduled purge using the same code path as erasure, fixed backup rotation. |
| Processor agreement | Art. 28; EDPB 07/2020 | Accept the VPS host's DPA, list sub-processors. |
| No transfers | Art. 44 to 49; EDPB 05/2021 | EU host and EU data centre for every service that touches personal data. |
| Security | Art. 32; Telegram §4.4 | Encryption at rest with separate key, TLS, tested restore. |
| Breaches | Art. 33, 34 | Breach log, 72-hour runbook. |
| Records | Art. 30 | Record of processing in the repo. |
| DPIA | Art. 35; WP248 | Short DPIA, check the national list. |
| Joint controllers | Art. 26 | Written split of duties between the two of you. |

## Open questions for legal review

1. Is Telegram a recipient only, or does sending bot replies to Telegram Messenger Inc. count as a transfer outside the EEA? If it does, which Chapter V tool covers other Members' data in Group replies?
2. Is option B (placeholder in shared Groups) a lawful answer to an erasure request, and is the remaining Members' legitimate interest enough basis to keep the rows?
3. Which supervisory authority is competent? It depends on the Member State where the two of you are established, which this research does not know. That authority's Art. 35(4) DPIA list and breach form apply.
4. Can a Category like "Medical" make an Expense special category data under Art. 9 and C-252/21?
5. Does Telegram's consent clause in Bot Developer Terms §4.3 need anything beyond an acceptance step, given the GDPR basis is contract?
6. Minimum age: what national contract law says about minors accepting the terms.

## Sources

Primary law:

- Regulation (EU) 2016/679 (GDPR), consolidated text from the EU Publications Office: <https://eur-lex.europa.eu/eli/reg/2016/679/oj>. Articles 2, 3, 4, 5, 6, 8, 9, 12 to 21, 26, 27, 28, 30, 32 to 37, 44 to 49, Recitals 18, 26, 39, 65, 68.
- Commission Implementing Decision (EU) 2023/1795 on the EU-US Data Privacy Framework: <https://eur-lex.europa.eu/eli/dec_impl/2023/1795/oj>
- CJEU, Case C-252/21, Meta Platforms and Others v Bundeskartellamt, judgment of 4 July 2023: <https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:62021CJ0252>

EDPB and WP29 guidance endorsed by the EDPB:

- EDPB Guidelines 2/2019 on Article 6(1)(b) in online services, version 2.0, 8 October 2019: <https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-22019-processing-personal-data-under-article-61b_en>
- EDPB Guidelines 01/2022 on the right of access, version 2.1, adopted 28 March 2023: <https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-012022-data-subject-rights-right-access_en>
- EDPB Guidelines 07/2020 on controller and processor, version 2.1: <https://www.edpb.europa.eu/system/files/2023-10/EDPB_guidelines_202007_controllerprocessor_final_en.pdf>
- EDPB Guidelines 05/2021 on the interplay between Article 3 and Chapter V, version 2.0, 14 February 2023: <https://www.edpb.europa.eu/system/files/2023-02/edpb_guidelines_05-2021_interplay_between_the_application_of_art3-chapter_v_of_the_gdpr_v2_en_0.pdf>
- EDPB Guidelines 9/2022 on personal data breach notification, version 2.0, 28 March 2023: <https://www.edpb.europa.eu/system/files/2023-04/edpb_guidelines_202209_personal_data_breach_notification_v2.0_en.pdf>
- WP29 Guidelines on the right to data portability, WP242 rev.01, 5 April 2017: <https://www.edpb.europa.eu/documents/guideline/guidelines-on-the-right-to-data-portability-under-regulation-2016679-wp242_en>
- WP29 Guidelines on transparency, WP260 rev.01, 11 April 2018: <https://ec.europa.eu/newsroom/article29/items/622227>
- WP29 Guidelines on DPIA, WP248 rev.01, 4 October 2017: <https://ec.europa.eu/newsroom/article29/items/611236>
- WP29 position paper on the derogations from records of processing, Art. 30(5), 19 April 2018: <https://www.edpb.europa.eu/documents/other-guidance/position-paper-on-the-derogations-from-the-obligation-to-maintain-records_en>

National authority:

- CNIL, GDPR developer's guide, Sheet n°14, "Define a data retention period": <https://www.cnil.fr/en/sheet-ndeg14-define-data-retention-period>

Telegram (first-party terms):

- Telegram Privacy Policy, §1.4, §4.1, §6: <https://telegram.org/privacy>
- Telegram Bot Developer Terms of Service, §4, §4.2 to §4.4, §9.1: <https://telegram.org/tos/bot-developers>
- Telegram Standard Bot Privacy Policy: <https://telegram.org/privacy-tpa>
