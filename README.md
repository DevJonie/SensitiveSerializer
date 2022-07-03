# Sensitive Serializer

## Sample

```csharp
using System.Reflection;
using System.Text.Json;
using static System.Console;


var data = new Data
{
    NotSensitive = "NotSensitive-L1",
    Sensitive = "Sensitive data-L1",
    HasSecretData = new Data
    {
        NotSensitive = "NotSensitive-L2",
        Sensitive = "Sensitive data-L2",
        HasSecretData = new Data
        {
            NotSensitive = "NotSensitive-L3",
            Sensitive = "Sensitive data-L3",
        }
    }
};

WriteLine(data.Serialize());

WriteLine(data.SensitiveSerialize());

class Data
{
    public string NotSensitive { get; set; } = null!;

    [Sensitive]
    public string Sensitive { get; set; } = null!;

    public Data HasSecretData { get; set; } = null!;
}



//=======================================================================
[AttributeUsage(AttributeTargets.Property,
    Inherited = true, AllowMultiple = false)]
class SensitiveAttribute : Attribute { }

record SensitiveSerializerOptions(byte MaxDepth = 3);

static class SensitiveSerializerExtensions
{
    public static string Serialize<T>(
        this T @object,
        JsonSerializerOptions? JsonOptions = default)
        => JsonSerializer.Serialize(@object, JsonOptions);


    public static string SensitiveSerialize<T>(
        this T @object,
        SensitiveSerializerOptions? sensitiveOptions = default,
        JsonSerializerOptions? jsonOptions = default)
    {
        if (@object is null) return @object.Serialize(jsonOptions);

        sensitiveOptions = sensitiveOptions ?? new();

        SanitizeSensitiveProperties(@object, sensitiveOptions.MaxDepth);

        return @object.Serialize(jsonOptions);
    }


    public static T? Deserialize<T>(
        this string jsonString,
        JsonSerializerOptions? jsonOptions = default)
        => JsonSerializer.Deserialize<T>(jsonString, jsonOptions);


    private static void SanitizeSensitiveProperties<T>(T @object, int maxDepth)
    {
        if (@object is null || maxDepth <= 0) return;

        var properties = @object.GetType().GetProperties();

        foreach (var property in properties)
        {
            if (property.IsSensitive())
            {
                object? replaceWith = property.PropertyType == typeof(string)
                    ? "******" : default;

                property.SetValue(@object, replaceWith);

                continue;
            }
            if (property.PropertyType.IsSimpleType()) continue;

            var value = property.GetValue(@object);

            if (value is null || value == default) continue;

            SanitizeSensitiveProperties(value, maxDepth - 1);

            property.SetValue(@object, value);
        }
    }

    private static bool IsSensitive(this PropertyInfo property)
        => property.GetCustomAttribute<SensitiveAttribute>() != null;

    private static bool IsSimpleType(this Type type)
    {
        return
            type.IsEnum ||
            type.IsPrimitive ||
            !IsObjectTypeCode(type) ||
            IsSimpleGenericType(type) ||
            IsSimpleNullableType(type) ||
            SimpleTypesArray().Contains(type);

        static bool IsSimpleNullableType(Type type)
        {
            var underlyingType = Nullable.GetUnderlyingType(type);
            return underlyingType != null && IsSimpleType(underlyingType);
        }

        static bool IsSimpleGenericType(Type type)
        {
            return type.IsGenericType &&
                type.GetGenericTypeDefinition() == typeof(Nullable<>) &&
                IsSimpleType(type.GetGenericArguments()[0]);
        }

        static bool IsObjectTypeCode(Type type)
        {
            return Convert.GetTypeCode(type) == TypeCode.Object;
        }

        static Type[] SimpleTypesArray()
        {
            return new Type[] {
                typeof(string),
                typeof(decimal),
                typeof(DateTime),
                typeof(DateOnly),
                typeof(TimeOnly),
                typeof(DateTimeOffset),
                typeof(TimeSpan),
                typeof(Guid)
            };
        }

    }

}
```
