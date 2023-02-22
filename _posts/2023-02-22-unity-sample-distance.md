---
layout: single
title:  "Unity 조각코드 (시야각, 거리, 랜덤 오브젝트 검출및 생성)"
categories: "Unity"
tag: ["조각코드", "게임", "시야각"]
toc: true
author_profile: true
sidebar:
    nav: "docs"
---

게임 개발을 하다보면 자주 사용하지만 정리를 잘 못해서 항상 코딩을 하는 공통 코드들이 있습니다.
소스는 따로 정리한다고 해도 나중에 블로그에서도 바로 볼수 있도록 여기에 정리를 하겠습니다.

## 시야각과 특정 거리 오브젝트 추출
```csharp
using UnityEngine;

public class VisionCheck : MonoBehaviour
{
    public float visionAngle = 45.0f; // 시야각
    public float visionDistance = 10.0f; // 시야 거리
    public float detectionDistance = 2.0f; // 검출 거리

    void Update()
    {
        // 현재 오브젝트의 위치와 바라보는 방향을 구합니다.
        Vector3 origin = transform.position;
        Vector3 direction = transform.forward;

        // 현재 오브젝트가 바라보는 방향에서 시야각 범위 내에 있는 모든 콜라이더를 검출합니다.
        RaycastHit[] hits = Physics.RaycastAll(origin, direction, visionDistance);
        foreach (RaycastHit hit in hits)
        {
            // 검출된 콜라이더의 게임오브젝트를 가져옵니다.
            GameObject target = hit.collider.gameObject;

            // 현재 오브젝트와 검출된 오브젝트의 거리를 계산합니다.
            float distance = Vector3.Distance(target.transform.position, origin);

            // 일정 거리 이내에 있는 오브젝트인지 검사합니다.
            if (distance <= detectionDistance)
            {
                // 현재 오브젝트와 검출된 오브젝트의 각도를 계산합니다.
                Vector3 targetDirection = target.transform.position - origin;
                float angle = Vector3.Angle(targetDirection, direction);

                // 계산된 각도가 시야각 범위 내에 있는지 검사합니다.
                if (angle <= visionAngle * 0.5f)
                {
                    Debug.Log("Detected target: " + target.name);
                }
            }
        }
    }
}
```
일정 거리 이내에 있는 오브젝트를 검출합니다. 검출된 오브젝트와 현재 오브젝트와의 거리를 계산하여, 일정 거리 이내에 있는 오브젝트인 경우에는 Vector3.Angle 메서드를 사용하여 시야각 범위 내에 있는지도 검사합니다. 

## 랜덤 원안에 오브젝트 생성
적 생성시 자주사용 될 수 있음
```csharp
using UnityEngine;

public class ObjectSpawner : MonoBehaviour
{
    public GameObject objectPrefab; // 생성할 오브젝트의 프리팹
    public float spawnRadius = 5.0f; // 생성 범위 반경

    public int maxObjects = 10; // 생성할 최대 오브젝트 수
    private int spawnedObjects = 0; // 생성된 오브젝트 수

    private void Start()
    {
        // 최대 생성 오브젝트 수에 도달할 때까지 오브젝트를 생성합니다.
        while (spawnedObjects < maxObjects)
        {
            SpawnObject();
        }
    }

    private void SpawnObject()
    {
        // 생성 위치를 랜덤하게 결정합니다.
        Vector3 spawnPosition = transform.position + Random.insideUnitSphere * spawnRadius;

        // 생성할 오브젝트를 인스턴스화합니다.
        GameObject newObject = Instantiate(objectPrefab, spawnPosition, Quaternion.identity);

        // 생성된 오브젝트 수를 증가시킵니다.
        spawnedObjects++;
    }
}
```
이 게임 오브젝트 주변에 지정한 반경 내에서 오브젝트를 생성합니다. 생성할 오브젝트의 프리팹은 objectPrefab 변수에 할당하며, 생성 범위 반경은 spawnRadius 변수로 지정합니다. 생성할 최대 오브젝트 수는 maxObjects 변수로 지정하며, 생성된 오브젝트 수는 spawnedObjects 변수로 카운트합니다.

2D 공간에서는 아래를 참고하여 응용 가능합니다.
```csharp
public GameObject objectPrefab;
public float radius = 5f; // 생성 범위의 반지름
public Vector2 centerPosition; // 생성 범위의 중심 위치

void Start()
{
    Vector2 spawnPosition = Random.insideUnitCircle * radius + centerPosition;
    Instantiate(objectPrefab, spawnPosition, Quaternion.identity);
}
```

## 랜덤 사각형 안에 오브젝트 생성

3D 기준 아래처럼 사용할수 있습니다.
```csharp
using UnityEngine;

public class ObjectSpawner : MonoBehaviour
{
    public GameObject objectPrefab; // 생성할 오브젝트의 프리팹

    public float spawnWidth = 5.0f; // 생성 범위의 가로 길이
    public float spawnHeight = 5.0f; // 생성 범위의 세로 길이

    public int maxObjects = 10; // 생성할 최대 오브젝트 수
    private int spawnedObjects = 0; // 생성된 오브젝트 수

    private void Start()
    {
        // 최대 생성 오브젝트 수에 도달할 때까지 오브젝트를 생성합니다.
        while (spawnedObjects < maxObjects)
        {
            SpawnObject();
        }
    }

    private void SpawnObject()
    {
        // 생성 위치를 랜덤하게 결정합니다.
        float spawnX = Random.Range(-spawnWidth / 2.0f, spawnWidth / 2.0f);
        float spawnY = Random.Range(-spawnHeight / 2.0f, spawnHeight / 2.0f);
        Vector3 spawnPosition = transform.position + new Vector3(spawnX, 0.0f, spawnY);

        // 생성할 오브젝트를 인스턴스화합니다.
        GameObject newObject = Instantiate(objectPrefab, spawnPosition, Quaternion.identity);

        // 생성된 오브젝트 수를 증가시킵니다.
        spawnedObjects++;
    }

    private void OnDrawGizmosSelected()
    {
        // 생성 범위를 시각적으로 표시합니다.
        Gizmos.color = Color.green;
        Gizmos.DrawWireCube(transform.position, new Vector3(spawnWidth, 0.0f, spawnHeight));
    }
}
```
 ObjectSpawner 스크립트가 붙어 있는 게임오브젝트를 중심으로 지정한 가로 길이와 세로 길이를 가지는 사각형 범위 내에서 오브젝트를 생성합니다. 생성할 오브젝트의 프리팹은 objectPrefab 변수에 할당하며, 생성 범위의 가로 길이와 세로 길이는 각각 spawnWidth와 spawnHeight 변수로 지정합니다. 생성할 최대 오브젝트 수는 maxObjects 변수로 지정하며, 생성된 오브젝트 수는 spawnedObjects 변수로 카운트합니다.

 OnDrawGizmosSelected() 함수는 에디터에서 해당 스크립트를 가진 게임오브젝트를 선택하고, 해당 함수를 구성한 경우 생성 범위를 시각적으로 표시하기 위한 함수입니다. 생성 범위를 녹색의 와이어프레임으로 그리도록 구현되어 있으며, 에디터 모드에서만 호출됩니다.

2D공간 아래소스 참고해서 응용 가능합니다.
```csharp
public GameObject objectPrefab;
public float rangeX = 10f; // 생성 범위의 X축 길이
public float rangeY = 10f; // 생성 범위의 Y축 길이

void Start()
{
    Vector2 spawnPosition = new Vector2(Random.Range(-rangeX, rangeX), Random.Range(-rangeY, rangeY));
    Instantiate(objectPrefab, spawnPosition, Quaternion.identity);
}
```
