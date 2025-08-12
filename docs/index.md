## Section 1

```csharp



    class Program
    {
        static void Main(string[] args)
        {
            // === Settings (modify as needed) ===
            int numLinesToGrab = 2000;                        // Number of lines to acquire (N)
            string savePath = @"C:\Images\linescan_result.png";
            string deviceUserID = "0";                        // First found device or Camera UserID
            string triggerSource = "Line1";                   // External trigger input line
            string triggerActivation = "RisingEdge";          // Rising/Falling edge
            int grabTimeoutMs = -1;                           // -1: Wait indefinitely (depending on model, "grab_timeout" may also be used)
            bool enforceSameWidth = true;                     // Ensure same width for safety (should be same width basically)

            HTuple acqHandle = null;
            var lineImages = new List<HObject>(capacity: numLinesToGrab);

            try
            {
                acqHandle = OpenGigEVision(deviceUserID);

                // LineStart (1 pulse = 1 line acquisition)
                ConfigureLineTrigger(acqHandle, triggerSource, triggerActivation);

                // Specify pixel format if needed (e.g., monochrome)
                // HOperatorSet.SetFramegrabberParam(acqHandle, "PixelFormat", "Mono8");

                HOperatorSet.GrabImageStart(acqHandle, -1);

                // Acquire N lines (returns one line per external trigger)
                for (int i = 0; i < numLinesToGrab; i++)
                {
                    HObject oneLine = GrabOneLine(acqHandle, grabTimeoutMs);
                    lineImages.Add(oneLine);
                    if ((i + 1) % 100 == 0) Console.WriteLine($"Grabbed lines: {i + 1}/{numLinesToGrab}");
                }

                // Option to align widths for safety (should be same width basically)
                if (enforceSameWidth) EqualizeWidthsInPlace(lineImages);

                // Combine vertically (N rows x 1 column) to create 2D image
                HObject fullImage = TileImagesVertically(lineImages);

                Directory.CreateDirectory(Path.GetDirectoryName(savePath) ?? ".");
                HOperatorSet.WriteImage(fullImage, GetExtAsHalconType(savePath), 0, savePath);
                Console.WriteLine($"Saved: {savePath}");

                fullImage.Dispose();
            }
            catch (HalconException hex)
            {
                Console.Error.WriteLine($"HALCON ERROR: #{hex.GetErrorCode()} {hex.GetErrorMessage()}");
            }
            catch (Exception ex)
            {
                Console.Error.WriteLine($"ERROR: {ex.Message}");
            }
            finally
            {
                foreach (var img in lineImages) img?.Dispose();
                if (acqHandle != null && acqHandle.IsInitialized())
                {
                    try { HOperatorSet.CloseFramegrabber(acqHandle); } catch { /* ignore */ }
                }
            }
        }

        // Open GigE Vision frame grabber
        static HTuple OpenGigEVision(string deviceUserID)
        {
            HOperatorSet.OpenFramegrabber(
                "GigEVision",
                0, 0, 0, 0, 0, 0,
                "progressive",
                -1,
                "default",
                -1,
                "false",      // External trigger details set below
                "default",
                deviceUserID, // "0" or Camera UserID
                0,
                -1,
                out HTuple acqHandle);

            // Set bandwidth etc. here if needed (model dependent)
            // HOperatorSet.SetFramegrabberParam(acqHandle, "GevSCPSPacketSize", 9000);

            return acqHandle;
        }

        // Set line trigger (GenICam keys are model dependent. General example)
        static void ConfigureLineTrigger(HTuple acqHandle, string triggerSource, string triggerActivation)
        {
            // Which event to trigger on: line unit
            HOperatorSet.SetFramegrabberParam(acqHandle, "TriggerSelector", "LineStart");

            // Enable trigger
            HOperatorSet.SetFramegrabberParam(acqHandle, "TriggerMode", "On");

            // External input line or encoder etc. (e.g., Line1)
            HOperatorSet.SetFramegrabberParam(acqHandle, "TriggerSource", triggerSource);

            // Rising/Falling edge
            HOperatorSet.SetFramegrabberParam(acqHandle, "TriggerActivation", triggerActivation);

            // Set exposure or gain if needed
            // HOperatorSet.SetFramegrabberParam(acqHandle, "ExposureTime", 20_000.0); // Example: 20,000 us
            // For encoder synchronization, set model-specific features (e.g., LineTriggerSource="Encoder" etc.)
        }

        // Acquire one line (blocks until external trigger arrives)
        static HObject GrabOneLine(HTuple acqHandle, int timeoutMs)
        {
            // The third argument of GrabImageAsync is -1 for infinite wait. If you want to use timeout, set "grab_timeout" if supported by the model
            HOperatorSet.GrabImageAsync(out HObject lineImg, acqHandle, -1);
            return lineImg;
        }


        static HObject GrabOneLineOrBlack(HTuple acqHandle, int timeoutMs, int fallbackWidth = 4096, string pixelType = "byte")
        {
            try
            {
                HOperatorSet.GrabImageAsync(out HObject lineImg, acqHandle, -1);
                return lineImg;
            }
            catch (HalconException ex)
            {
                HOperatorSet.GenImageConst(out HObject blankLine, pixelType, fallbackWidth, 1);
                return blankLine;
            }
        }



        // Combine images vertically (N rows x 1 column)
        static HObject TileImagesVertically(List<HObject> imageList)
        {
            if (imageList == null || imageList.Count == 0)
                throw new ArgumentException("imageList is empty.");

            HOperatorSet.GenEmptyObj(out HObject all);
            foreach (var img in imageList)
            {
                HOperatorSet.ConcatObj(all, img, out all);
            }

            HOperatorSet.TileImages(all, out HObject tiled, imageList.Count, 1);
            all.Dispose();
            return tiled;
        }

        // Align widths to the maximum (should be same width basically, but for safety)
        static void EqualizeWidthsInPlace(List<HObject> imageList)
        {
            int maxWidth = 0;
            var sizes = new List<(int w, int h)>(imageList.Count);

            foreach (var img in imageList)
            {
                HOperatorSet.GetImageSize(img, out HTuple w, out HTuple h);
                int wi = w.I, hi = h.I;         // For line, h is usually 1
                sizes.Add((wi, hi));
                if (wi > maxWidth) maxWidth = wi;
            }

            for (int i = 0; i < imageList.Count; i++)
            {
                var (w, h) = sizes[i];
                if (w == maxWidth) continue;

                double scale = (double)maxWidth / Math.Max(1, w);
                int newH = Math.Max(1, (int)Math.Round(h * scale)); // h=1 is usual
                HOperatorSet.ZoomImageSize(imageList[i], out HObject resized, maxWidth, newH, "constant");
                imageList[i].Dispose();
                imageList[i] = resized;
            }
        }

        // Extension -> HALCON write type
        static string GetExtAsHalconType(string path)
        {
            var ext = Path.GetExtension(path)?.ToLowerInvariant();
            return ext switch
            {
                ".png" => "png",
                ".jpg" or ".jpeg" => "jpeg",
                ".bmp" => "bmp",
                ".tif" or ".tiff" => "tiff",
                _ => "png"
            };
        }
    }
