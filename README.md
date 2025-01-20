# インターネットコンピュータ上で分散型クラウドファンディングDAppを構築

インターネットコンピュータ（ICP）は、従来のインターネットの限界を超えた分散型エコシステムを提供するブロックチェーンプロジェクトです。その目標は、Web 3.0を加速させ、クラウドサービスに依存せずに自己完結型のシステムを可能にすることです。特に、スマートコントラクトを活用して完全に分散化されたアプリケーション（DApp）を構築できる点が特徴です。

## **ICPの特長**

1. **スケーラビリティ**: ICPはノードを追加することでスケーラブルに拡張可能であり、アプリケーションの成長に柔軟に対応できます。
2. **低コスト**: ICPの計算コストは従来のクラウドサービスに比べて低く抑えられています。
3. **高速トランザクション**: トランザクションは数秒で確定し、リアルタイムのユーザー体験を提供します。
4. **分散型のセキュリティ**: データとロジックが完全に分散化されているため、改ざん耐性が高くセキュアです。
5. **開発者向けのツール**: Azleのようなフレームワークにより、TypeScriptを使用して効率的にスマートコントラクトを構築できます。

この記事では、[Azle](https://github.com/demergent-labs/azle) を使用して構築された分散型クラウドファンディングアプリケーション (DApp) について説明します。AzleはTypeScriptでインターネットコンピュータ (ICP) のキャニスターを構築するためのフレームワークです。このDAppは、ユーザーがクラウドファンディングキャンペーンを作成・管理し、貢献し、ユーザープロファイルをシームレスに管理できるようにします。以下では、コードの主要コンポーネントと機能について説明します。

## 主な機能
1. **ユーザー登録**: ユニークな名前と初期残高で登録可能。
2. **キャンペーン管理**:
   - 新しいキャンペーンの作成。
   - キャンペーンの詳細を更新。
   - キャンペーンを終了 (成功またはキャンセルとしてマーク)。
3. **貢献システム**:
   - アクティブなキャンペーンに貢献。
   - キャンペーンがキャンセルされた場合の返金。
4. **検索と統計**:
   - キーワードでキャンペーンを検索。
   - 総資金調達額、貢献者数、平均貢献額などのキャンペーン統計を取得。

---

## コードの解説

### **データ構造の定義**

このアプリケーションでは、データをモデル化するために`Variant`と`Record`型を使用します。

- **キャンペーンステータス**:
  ```typescript
  const CampaignStatus = Variant({
      Active: text,
      Completed: text,
      Cancelled: text
  });
  ```
- **キャンペーンと貢献**:
  ```typescript
  const Campaign = Record({
      id: text,
      title: text,
      description: text,
      targetAmount: nat64,
      currentAmount: nat64,
      beneficiary: Principal,
      status: CampaignStatus,
      creationDate: nat64,
      endDate: nat64
  });

  const CampaignContribution = Record({
      campaignId: text,
      contributor: Principal,
      amount: nat64,
      timestamp: nat64
  });
  ```
- **ユーザープロファイル**:
  ```typescript
  const UserProfile = Record({
      principal: Principal,
      name: text,
      balance: nat64
  });
  ```

### **永続ストレージ**

`StableBTreeMap`を使用してデータを永続化します:
```typescript
const campaignsStorage = StableBTreeMap(0, text, Campaign);
const contributionsStorage = StableBTreeMap(1, text, Vec(CampaignContribution));
const userProfiles = StableBTreeMap(2, Principal, UserProfile);
```
これらのマップはキャンペーン、貢献、ユーザープロファイルを保存します。

### **ユーザー登録**

ユーザーは名前と初期残高で登録できます:
```typescript
registerUser: update([text, nat64], Result(text, Message), (name, initialBalance) => {
    const userPrincipal = ic.caller();
    const userProfile = {
        principal: userPrincipal,
        name: name,
        balance: initialBalance
    };
    userProfiles.insert(userPrincipal, userProfile);
    return Ok(`User ${name} registered successfully with principal ${userPrincipal.toText()} and initial balance ${initialBalance}`);
})
```

### **キャンペーン作成**

ユーザーはタイトル、説明、目標金額、終了日時を指定して新しいキャンペーンを作成できます:
```typescript
createCampaign: update([text, text, nat64, nat64], Result(text, Message), async (title, description, targetAmount, endDate) => {
    const campaignId = uuidv4();
    const campaign = {
        id: campaignId,
        title,
        description,
        targetAmount,
        currentAmount: 0n,
        beneficiary: ic.caller(),
        status: {Active: "ACTIVE"},
        creationDate: ic.time(),
        endDate
    };
    campaignsStorage.insert(campaignId, campaign);
    return Ok(campaignId);
})
```

### **キャンペーンへの貢献**

貢献は、貢献者の残高から金額を差し引き、キャンペーンの現在の金額を更新することで処理されます:
```typescript
contributeToCampaign: update([text, nat64], Result(text, Message), async (campaignId, amount) => {
    const campaignOpt = campaignsStorage.get(campaignId);
    const campaign = campaignOpt.Some;

    const userOpt = userProfiles.get(ic.caller());
    const user = userOpt.Some;

    if (user.balance < amount) {
        return Err({ InvalidPayload: `Insufficient balance` });
    }

    campaign.currentAmount += amount;
    user.balance -= amount;

    campaignsStorage.insert(campaignId, campaign);
    userProfiles.insert(ic.caller(), user);

    const contribution = {
        campaignId,
        contributor: ic.caller(),
        amount,
        timestamp: ic.time()
    };
    let contributions = [];
    const contributionsOpt = contributionsStorage.get(campaignId);
    if ("Some" in contributionsOpt) {
        contributions = contributionsOpt.Some;
    }
    contributions.push(contribution);
    contributionsStorage.insert(campaignId, contributions);

    return Ok(`Contribution of ${amount} ICP to campaign ${campaignId} successful.`);
})
```

### **キャンペーンの終了**

キャンペーンはそのステータスに基づいて終了できます:
```typescript
closeCampaign: update([text], Result(text, Message), async (campaignId) => {
    const campaignOpt = campaignsStorage.get(campaignId);
    const campaign = campaignOpt.Some;

    if (campaign.currentAmount >= campaign.targetAmount) {
        campaign.status = {Completed: "COMPLETED"};
    } else {
        campaign.status = {Cancelled: "CANCELLED"};
        const contributions = contributionsStorage.get(campaignId).Some;
        for (const contribution of contributions) {
            const userOpt = userProfiles.get(contribution.contributor);
            const user = userOpt.Some;
            user.balance += contribution.amount;
            userProfiles.insert(contribution.contributor, user);
        }
    }
    campaignsStorage.insert(campaignId, campaign);
    return Ok(`Campaign ${campaignId} closed successfully.`);
})
```

### **キャンペーンの検索と統計**

キーワードでキャンペーンを検索:
```typescript
searchCampaigns: query([text], Vec(Campaign), (keyword) => {
    const campaigns = campaignsStorage.values();
    return campaigns.filter(campaign =>
        campaign.title.includes(keyword) ||
        campaign.description.includes(keyword)
    );
})
```
キャンペーンの統計を取得:
```typescript
getCampaignStatistics: query([text], Record({
    totalAmountRaised: nat64,
    numberOfContributors: nat64,
    averageContribution: nat64
}), (campaignId) => {
    const contributions = contributionsStorage.get(campaignId).Some;
    const totalAmountRaised = contributions.reduce((total, contribution) => total + contribution.amount, 0n);
    const numberOfContributors = contributions.length;
    const averageContribution = numberOfContributors === 0 ? 0n : totalAmountRaised / BigInt(numberOfContributors);

    return {
        totalAmountRaised,
        numberOfContributors,
        averageContribution
    };
})
```

---

## 結論

この分散型クラウドファンディングアプリケーションは、インターネットコンピュータとAzleフレームワークの強力さを示しています。`StableBTreeMap`を活用することで、永続的で効率的なデータストレージを確保し、モジュール型アーキテクチャにより拡張性を持たせています。

このDAppを構築してデプロイすることで、分散化のメリットを直接体験してください。コーディングをお楽しみください！

