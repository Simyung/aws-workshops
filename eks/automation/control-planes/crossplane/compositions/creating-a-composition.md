# Creating a Composition

`CompositeResourceDefinition` (XRD)는 Composite Resource (XR)의 유형과 스키마를 정의합니다. XRD는 원하는 XR과 그 필드에 대해 Crossplane에 알려줍니다. XRD는 CustomResourceDefinition (CRD)과 유사하지만 더 체계적인 구조를 가지고 있습니다. XRD를 생성하는 것은 주로 OpenAPI "[structural schema](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)"를 지정하는 것을 포함합니다.

애플리케이션 팀 구성원들이 각자의 네임스페이스에서 DynamoDB 테이블을 생성할 수 있도록 하는 정의부터 시작해보겠습니다. 이 예제에서 사용자들은 이름, 키 속성, 인덱스 이름 필드만 지정하면 됩니다.

{% code title="~/environment/eks-workshop/modules/automation/controlplanes/crossplane/compositions/composition/definition.yaml" %}
```
# Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.
# SPDX-License-Identifier: Apache-2.0

apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdynamodbtables.awsblueprints.io
spec:
  group: awsblueprints.io
  names:
    kind: XDynamoDBTable
    plural: xdynamodbtables
  claimNames:
    kind: DynamoDBTable
    plural: dynamodbtables
  connectionSecretKeys:
    - tableName
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          description: Table is the Schema for the tables API
          properties:
            spec:
              type: object
              properties:
                resourceConfig:
                  properties:
                    deletionPolicy:
                      description: Defaults to Delete
                      enum:
                        - Delete
                        - Orphan
                      type: string
                    name:
                      type: string
                    providerConfigName:
                      type: string
                      default: aws-provider-config
                    region:
                      type: string
                      default: ""
                    tags:
                      additionalProperties:
                        type: string
                      description: Key-value map of resource tags.
                      type: object
                  required:
                    - region
                  type: object
                dynamoConfig:
                  properties:
                    attribute: #required for hashKey and/or rangeKey
                      items:
                        properties:
                          name: #name of the hashKey and/or rangeKey
                            type: string
                          type:
                            enum:
                              - B #binary
                              - N #number
                              - S #string
                            type: string
                        required:
                          - name
                          - type
                        type: object
                      type: array
                    hashKey:
                      type: string
                    rangeKey:
                      type: string
                    billingMode:
                      type: string
                      default: PAY_PER_REQUEST
                    readCapacity:
                      type: number
                    writeCapacity:
                      type: number
                    globalSecondaryIndex:
                      items:
                        properties:
                          hashKey:
                            type: string
                          name:
                            type: string
                          rangeKey:
                            type: string
                          readCapacity:
                            type: number
                          writeCapacity:
                            type: number
                          projectionType:
                            type: string
                            default: ALL
                          nonKeyAttributes: #required for gsi
                            items:
                              type: string
                            type: array
                        type: object
                        required:
                          - name
                      type: array
                    localSecondaryIndex:
                      items:
                        properties:
                          name:
                            type: string
                          rangeKey:
                            type: string
                          projectionType:
                            type: string
                          nonKeyAttributes: #required for lsi
                            items:
                              type: string
                            type: array
                        type: object
                        required:
                          - name
                          - rangeKey
                          - projectionType
                          - nonKeyAttributes
                      type: array
                  required:
                    - attribute
                  type: object
              required:
                - dynamoConfig
            status:
              type: object
              description: TableStatus defines the observed state of Table
              properties:
                tableArn:
                  description: Indicates this table's ARN
                  type: string
                tableName:
                  description: Indicates this table's Name
                  type: string
          required:
            - spec

```
{% endcode %}

Composition은 Composite Resource가 생성될 때 취해야 할 조치에 대해 Crossplane에 알려줍니다. 각 Composition은 XR과 하나 이상의 Managed Resources 집합 간의 연결을 설정합니다. XR이 생성, 업데이트 또는 삭제될 때, 연관된 Managed Resources도 그에 따라 생성, 업데이트 또는 삭제됩니다.

다음 Composition은 managed resource `Table`을 프로비저닝합니다:

{% code title="~/environment/eks-workshop/modules/automation/controlplanes/crossplane/compositions/composition/table.yaml" %}
```
# Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.
# SPDX-License-Identifier: Apache-2.0

apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: table.dynamodb.awsblueprints.io
  labels:
    awsblueprints.io/provider: aws
    awsblueprints.io/environment: dev
spec:
  writeConnectionSecretsToNamespace: crossplane-system
  compositeTypeRef:
    apiVersion: awsblueprints.io/v1alpha1
    kind: XDynamoDBTable
  patchSets:
    - name: common-fields
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.resourceConfig.providerConfigName
          toFieldPath: spec.providerConfigRef.name
        - type: FromCompositeFieldPath
          fromFieldPath: spec.name
          toFieldPath: metadata.annotations[crossplane.io/external-name]
        - type: FromCompositeFieldPath
          fromFieldPath: metadata.name
          toFieldPath: metadata.annotations[crossplane.io/external-name]
          transforms:
            - type: string
              string:
                type: Regexp
                regexp:
                  match: ^(.*?)-crossplane
  resources:
    - name: table
      connectionDetails:
        - type: FromFieldPath
          name: tableName
          fromFieldPath: status.atProvider.id
      base:
        apiVersion: dynamodb.aws.upbound.io/v1beta1
        kind: Table
        spec:
          forProvider:
            writeConnectionSecretToRef:
              name: cartsdynamo
              namespace: crossplane-system
            region: ""
          providerConfigRef:
            name: aws-provider-config
      patches:
        - type: PatchSet
          patchSetName: common-fields
        - type: FromCompositeFieldPath
          fromFieldPath: spec.dynamoConfig.attribute
          toFieldPath: spec.forProvider.attribute
          policy:
            mergeOptions:
              appendSlice: true
              keepMapValues: true
        - type: FromCompositeFieldPath
          fromFieldPath: spec.resourceConfig.tags
          toFieldPath: spec.forProvider.tags
          policy:
            mergeOptions:
              keepMapValues: true
        - type: FromCompositeFieldPath
          fromFieldPath: spec.dynamoConfig.attribute[0].name
          toFieldPath: spec.forProvider.hashKey
        - type: FromCompositeFieldPath
          fromFieldPath: spec.dynamoConfig.billingMode
          toFieldPath: spec.forProvider.billingMode
        - type: FromCompositeFieldPath
          fromFieldPath: spec.dynamoConfig.rangeKey
          toFieldPath: spec.forProvider.rangeKey
        - type: FromCompositeFieldPath
          fromFieldPath: spec.dynamoConfig.readCapacity
          toFieldPath: spec.forProvider.readCapacity
        - type: FromCompositeFieldPath
          fromFieldPath: spec.dynamoConfig.writeCapacity
          toFieldPath: spec.forProvider.writeCapacity
        - type: FromCompositeFieldPath
          fromFieldPath: spec.dynamoConfig.globalSecondaryIndex[0].name
          toFieldPath: spec.forProvider.globalSecondaryIndex[0].name
        - type: FromCompositeFieldPath
          fromFieldPath: spec.dynamoConfig.attribute[1].name
          toFieldPath: spec.forProvider.globalSecondaryIndex[0].hashKey
        - type: FromCompositeFieldPath
          fromFieldPath: spec.dynamoConfig.globalSecondaryIndex[0].projectionType
          toFieldPath: spec.forProvider.globalSecondaryIndex[0].projectionType
          policy:
            mergeOptions:
              keepMapValues: true
        - type: FromCompositeFieldPath
          fromFieldPath: spec.dynamoConfig.localSecondaryIndex
          toFieldPath: spec.forProvider.localSecondaryIndex
          policy:
            mergeOptions:
              keepMapValues: true
        - type: ToCompositeFieldPath
          fromFieldPath: status.atProvider.id
          toFieldPath: status.tableName
        - type: ToCompositeFieldPath
          fromFieldPath: status.atProvider.arn
          toFieldPath: status.tableArn
```
{% endcode %}

이 구성을 EKS 클러스터에 적용해 보겠습니다:

```
~$ kubectl apply -k ~/environment/eks-workshop/modules/automation/controlplanes/crossplane/compositions/composition
compositeresourcedefinition.apiextensions.crossplane.io/xdynamodbtables.awsblueprints.io created
composition.apiextensions.crossplane.io/table.dynamodb.awsblueprints.io created
```

이러한 리소스들이 준비되면서, DynamoDB 테이블을 생성하기 위한 Crossplane Composition 설정이 성공적으로 완료되었습니다. 이러한 추상화를 통해 애플리케이션 개발자들은 기본 AWS 관련 세부사항을 이해할 필요 없이 표준화된 DynamoDB 테이블을 프로비저닝할 수 있습니다.

