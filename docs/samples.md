## Section 2

```csharp

using System;
using System.Collections.Generic;
using System.IO;
using HalconDotNet;

public static class HalconCameraConfig
{
    /// <summary>
    /// 2行CSV (ヘッダ + 値) からカメラパラメータを設定
    /// </summary>
    public static void ApplyCameraParamsFromCsv2Row(HTuple acqHandle, string csvFilePath)
    {
        if (!File.Exists(csvFilePath))
            throw new FileNotFoundException("CSVファイルが見つかりません", csvFilePath);

        var lines = File.ReadAllLines(csvFilePath);
        if (lines.Length < 2)
            throw new FormatException("CSVファイルはヘッダと値の2行が必要です");

        var headers = lines[0].Split(',');
        var values = lines[1].Split(',');

        if (headers.Length != values.Length)
            throw new FormatException("ヘッダと値の数が一致しません");

        for (int i = 0; i < headers.Length; i++)
        {
            string key = headers[i].Trim();
            string valStr = values[i].Trim();

            if (string.IsNullOrEmpty(key))
                continue;

            HTuple val;
            if (double.TryParse(valStr, out double d))
                val = d;
            else if (int.TryParse(valStr, out int n))
                val = n;
            else
                val = valStr;

            try
            {
                HOperatorSet.SetFramegrabberParam(acqHandle, key, val);
                Console.WriteLine($"Set: {key} = {val}");
            }
            catch (HalconException ex)
            {
                Console.WriteLine($"SetFramegrabberParam失敗: {key} = {val} → {ex.Message}");
            }
        }
    }
}
