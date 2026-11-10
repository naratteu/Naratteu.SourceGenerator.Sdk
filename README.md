# Naratteu.SourceGenerator.Sdk

소스제너레이터 프로젝트의 뻔한 설정을 담은 MSBuild SDK.

```xml
<Project Sdk="Naratteu.SourceGenerator.Sdk/0.0.14">
</Project>
```

FBA 로도 쓸 수 있습니다.

```csharp
#:sdk Naratteu.SourceGenerator.Sdk@0.0.14

using Microsoft.CodeAnalysis;

[Generator]
public class MyGenerator : IIncrementalGenerator { /* .. */ }
```

대신 해주는 것:

- `netstandard2.0` · `LangVersion 14` · `IsRoslynComponent` · `EnforceExtendedAnalyzerRules`
- `Microsoft.CodeAnalysis.CSharp` 와 `PolySharp` 참조
- `PackageType=Analyzer` 로 패킹. 생성기 dll 과 `@(Reference)` 로 끼워넣은 dll 들을
  **같은 디렉터리**(`analyzers/dotnet/cs/`)에 넣습니다

## Roslyn 버전은 낮게 잡는다

```xml
<RoslynVersion>4.8.0</RoslynVersion>  <!-- 기본값 -->
```

일반 라이브러리와 직관이 반대입니다. 분석기가 참조하는 Roslyn 버전은 **쓰는 사람의
컴파일러에 요구하는 최소 버전** 이 됩니다. 더 높은 걸 참조하면 그보다 낮은 컴파일러가
분석기를 통째로 건너뜁니다.

```
warning CS9057: Analyzer assembly 'x.dll' cannot be used because it references version
'5.3.0.0' of the compiler, which is newer than the currently running version '4.14.0.0'.
```

그러면 생성기가 안 도니까 쓰는 쪽은 `error CS0246` 만 보게 됩니다. 원인을 짚기 어렵습니다.

| 참조하는 Roslyn | 쓸 수 있는 SDK |
| --- | --- |
| `4.8` | .NET 8 SDK 이상 |
| `4.14` | .NET 9 SDK (9.0.3xx) 이상 |
| `5.3` | .NET 10 SDK 이상 |

Roslyn 5 의 API 가 꼭 필요하면 프로젝트에서 올리면 됩니다.

```xml
<RoslynVersion>5.3.0</RoslynVersion>
```

## 함정: `SyntaxList<T>` 에 컬렉션 표현식을 쓰지 말 것

`LangVersion 14` 로 쓰다 보면 무심코 쓰게 되는데, **Roslyn 5 미만에서는 컴파일도 되고
경고도 없이 빈 목록이 됩니다.** `SyntaxList<T>` · `SyntaxTokenList` 에 `[CollectionBuilder]`
가 붙은 것이 Roslyn 5 부터라서, 그 전에는 컬렉션 표현식이 비변경 `Add` 의 반환값을 버립니다.

```csharp
// Roslyn 4.x 에서 조용히 빈 목록이 된다
node.WithMembers([member])
    .WithModifiers([SyntaxFactory.Token(SyntaxKind.PartialKeyword)])

// 이렇게 쓴다
node.WithMembers(SyntaxFactory.SingletonList(member))
    .WithModifiers(SyntaxFactory.TokenList(SyntaxFactory.Token(SyntaxKind.PartialKeyword)))
```

멤버와 수정자가 통째로 사라지므로, 증상은 보통 엉뚱한 자리에서 터집니다.

## 함정: `SourceText.From` 의 `canBeEmbedded`

SDK 기본값인 `EmbedUntrackedSources` 가 생성 소스를 PDB 에 묻으려 할 때, Roslyn 4.x 에서는
**컴파일러가 죽습니다**.

```
System.ArgumentException: SourceText cannot be embedded. Provide encoding or canBeEmbedded=true at construction.
```

```csharp
SourceText.From(buffer, buffer.Length, Encoding.UTF8, canBeEmbedded: true)
```
