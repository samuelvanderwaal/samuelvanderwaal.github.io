---
layout: post
title: Rust vs Python
subtitle: Things aren't always as simple as I'd like.
tags: [softare, rust, python]
---

I've been playing around Rust for a few months now both to learn the language itself and as a way to broaden my programming meta-skills. Prior to this I've primarily worked with Python--a dynamic and duck-typed language with a garbage collector, and Rust offers a stark contrast with its strongly typed no-garbage collection design. Finally, knowing a performant language like Rust should provide more tools in the tool belt for solving problems that are processing intensive. 

I read through the Book--the canonical introduction to the Rust language and have implemented a handful of toy programs to learn the ropes. Recently, I successfully convinced a client to let me port over an unfinished project from Python to Rust in order to provide more performance. The project involves data parsing of largish logs of about 1+ GBs (serde, anyone?), using that parsed data to match images to powerline pole locations, parse smaller annotations logs (serde, anyone?) and crop hundreds to thousands of images with data from the annotations files. 

Today's post looks at the parallelization of image cropping and shows a performance comparison between my Python implementation and the Rust version. 

My first stab at the crop image function went for a serial approach as a sanity check that I had could actually manipulate images as expected.  It used a fixed crop size for simplicity and looked something like this:


```rust
pub fn crop_images(image_paths: glob::Paths) -> Result<(), Box<Error>> {
    for entry in image_paths {
        let path = entry?;
        crop_image(path, 500, 500, 100, 100)?;
    }
    Ok(())
}

fn crop_image(
    path: &std::path::PathBuf,
    x: u32,
    y: u32,
    width: u32,
    height: u32,
) -> Result<(), Box<Error>> {
    let parent = path.parent().unwrap();
    let name = path.file_name().unwrap();

    let ref mut img = image::open(&path)?;
    let subimg = imageops::crop(img, x, y, width, height);
    let save_path = parent.join(Path::new("cropped_images")).join(name);

    subimg.to_image().save(save_path)?;
    Ok(())
}
```

This uses the [image](https://crates.io/crates/image) crate for creating a sub-image of the crop size and then saving it as a new file. I haven't explored if there's a more performant approach for this but this works well enough for the time being. I also have some ugly `unwraps()` that probably need to be abstracted away into proper error handling once I figure out my approach for that with this project. 

This code works though and processed my test set of 287 JPGs in 3 minutes 52 seconds. 

Next, I wanted to figure out how to use the power of parallelization to speed up the processing by spreading it over multiple threads. My limited experience with Python and parallel processing involved setting up a worker group with Queue and pushing tasks to the queue where the workers can pull from. It looks like there are various ways to do this with Rust using `std::thread` but I've heard of this powerful parallelization library called `Rayon` so wanted to check it out. At it's simplest Rayon lets you convert your normal serial iteration to parallel by replacing `iter()` with its `par_iter()` function. I've heard this performance is excellent that that's almost too easy! 

In my current code, however, I'm not using the `iter()` function because `glob::glob()` returns an iterator already which I am passing directly to `crop_images()` and then using a for loop to iterate through. To use `rayon`'s `par_iter()` function I'll have to restructure my code a little:

```rust
pub fn crop_images(image_paths: glob::Paths) -> Result<(), Box<Error>> {
    // Collect elements into a vector for iteration.
    let col: Vec<Result<PathBuf, glob::GlobError>> = image_paths.collect();

    col.iter()
        .for_each(|path_result| match path_result {
            Ok(path) => match crop_image(path, 500, 500, 100, 100) {
                Ok(()) => (),
                Err(e) => println!("Cropping error: {:?}", e),
            },
            Err(e) => println!("Error: {:?}", e),
        }); 
    Ok(())
}
```

First I collect the glob-matched elements into a vector so I can iterate over them. `glob::glob()` returns a result with the `Ok` value being a `PathBuf` and the error value being a `glob:GlobError` so that's the type I tell the vector to expect:
` let col: Vec<Result<PathBuf, glob::GlobError>> = image_paths.collect();`

because `image_paths` is already an iterator I can just run `collect()` on it. Rust iterators are lazily evaluated so you need `collect()` to actually evaluate them. 

Then I iterate over the collection with `.iter()` and use `for_each()` and `match` statements to handle errors and apply the `crop_image()` function to each valid path value. 

This works and runs in approximately the same time as the for loop code. I didn't do more than one run through because of how long it takes but my run took 4 minutes which I think is roughly within the variance of what the for loop processing takes. 

Now it's a simple matter to change the `iter()` to `par_iter()` and rerun the code. At this point, I should note my performance runs are all with binaries generated by `cargo build --release`. 

```rust
  col.par_iter()
        .for_each(|path_result| match path_result {
            Ok(path) => match crop_image(path, 500, 500, 100, 100) {
                Ok(()) => (),
                Err(e) => println!("Cropping error: {:?}", e),
            },
            Err(e) => println!("Error: {:?}", e),
        }); 
    Ok(())
}
```

This completes in roughly 1 minute and 30 seconds and seems to make good use of all of my laptop's eight cores:

[![par_iter.png](https://svbtleusercontent.com/5DD2nCdiCk4MHJk8XYYzQv0xspap_small.png)](https://svbtleusercontent.com/5DD2nCdiCk4MHJk8XYYzQv0xspap.png)

My Python implementation uses a worker pool and 16 worker threads:

```python
import cv2
from glob import glob
from queue import Queue
from threading import Thread
from pathlib import Path


def main():
    image_dir_path = Path("/home/samuel/Desktop/TestData/cropped_images")
    image_paths = glob("/home/samuel/Desktop/TestData/*.JPG")

    for img in image_paths:
        name = Path(img).name
        item = (image_dir_path, Path(img), name, 500, 500, 100, 100)
        q.put(item)


def crop_image():
    while not q.empty():
        values = q.get()
        image_dir_path, image_path, name, x, y, width, height = values
        img = cv2.imread(str(image_path))
        print(f"cropping {name}...")
        try:
            cropped_img = img[y:y + height, x:x + width]
            cv2.imwrite(str(image_dir_path.joinpath(name)), cropped_img)
        except TypeError:
            print(f"type error, skpping image {name}")
            pass
        q.task_done()


if __name__ == "__main__":
    num_worker_threads = 16
    q = Queue()

    main()

    for i in range(num_worker_threads):
        t = Thread(target=crop_image)
        t.daemon = True
        t.start()

    q.join()
```

To my surprise this only takes about 50 seconds to run which is significantly faster than the Rust code. What's going on here? I think what's happening is that Python is actually calling opencv's C libraries for the image manipulation and so we're getting the blazing fast C speeds rather than a pure Python implementation. 

This is a bit disappointing, as I was expecting the Rust code to have a significant performance improvement over my Python implementation. At this point, the only way I can see of making a significant improvement in the Rust version is replacing the `image` library with a more performant one, perhaps trying out one of the OpenCV Rust bindings the community has created. That's a project for another day though.