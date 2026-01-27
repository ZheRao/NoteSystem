```python
import argparse
import datetime as dt

from path_adapter import run_so  # and later run_po, run_xx...


def main():
    p = argparse.ArgumentParser(prog="path_adapter")
    sub = p.add_subparsers(dest="pipeline", required=True)

    so = sub.add_parser("so", help="Run Sales Orders pipeline")
    so.add_argument("--date", default=None, help="YYYY-MM-DD (defaults to today)")
    so.add_argument("--input", default=None, help="Override PATH CSV filename")
    so.add_argument("--dry-run", action="store_true", help="Run without writing outputs")

    args = p.parse_args()

    run_date = dt.date.fromisoformat(args.date) if args.date else dt.date.today()

    if args.pipeline == "so":
        # Update run_so signature to accept these args, even if you ignore some for now
        run_so(run_date=run_date, input_file=args.input, dry_run=args.dry_run)


if __name__ == "__main__":
    main()
```